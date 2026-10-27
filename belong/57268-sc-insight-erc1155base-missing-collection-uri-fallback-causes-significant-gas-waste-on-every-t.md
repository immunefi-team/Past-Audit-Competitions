# 57268 sc insight erc1155base missing collection uri fallback causes significant gas waste on every token mint

**Submitted on Oct 24th 2025 at 20:37:12 UTC by @Happy\_Hunter for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57268
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/base/ERC1155Base.sol>
* **Impacts:** Operational gas inefficiency (higher mint gas costs when storing full per-token URIs)

## Description

### Brief / Intro

The `ERC1155Base.sol` contract stores token URIs individually and the `mint()` function documentation indicates the design is to set a "Token-specific URI to set (overrides collection URI)". While this provides per-token metadata control, it causes avoidable gas costs when many tokens share the same metadata pattern. Adopting an optional fallback pattern (like OpenZeppelin's `ERC1155URIStorage`) can avoid writing duplicate/empty URIs to storage and significantly reduce gas consumed by mint operations.

### Vulnerability Details

Affected contract:

* `contracts/v2/tokens/base/ERC1155Base.sol` (lines referenced: 88-96, 146-153)

Root cause (two compounding design choices):

1. No fallback logic: `uri(uint256 tokenId)` returns only the per-token `_tokenUri[tokenId]` and does not fall back to the collection-level `_uri` even when the per-token URI is empty.

```solidity
// ERC1155Base.sol lines 146-153
string private _uri;  // Collection-level base URI
mapping(uint256 tokenId => string tokenUri) private _tokenUri;  // Per-token overrides

function uri(uint256 tokenId) public view override returns (string memory) {
    return _tokenUri[tokenId];  // No fallback to _uri
}
```

2. Forced storage writes: `mint()` always writes the `tokenUri` parameter into storage (even when empty or redundant), causing expensive SSTORE operations for every mint.

```solidity
// ERC1155Base.sol lines 88-96
/// @notice Mints `amount` of `tokenId` to `to` and sets its token URI.
/// @param tokenUri Token-specific URI to set (overrides collection URI).
function mint(address to, uint256 tokenId, uint256 amount, string calldata tokenUri) 
    public onlyRole(MINTER_ROLE) 
{
    _setTokenUri(tokenId, tokenUri);  // ALWAYS writes to storage
    _mint(to, tokenId, amount, "0x");
}

function _setTokenUri(uint256 tokenId, string memory tokenUri) private {
    _tokenUri[tokenId] = tokenUri;  // SSTORE: expensive operation
    emit TokenUriSet(tokenId, tokenUri);
}
```

Gas impact analysis:

* Storing strings requires multiple storage slots (32 bytes each).
* A 40-character URI can consume \~3 storage slots.
* Each cold SSTORE (zero -> non-zero) costs \~20,000 gas.
* Rough total: \~60,000 gas for URI storage alone (approximate; varies by content and environment).

Comparison with OpenZeppelin: OpenZeppelin's `ERC1155URIStorage.sol` implements a fallback pattern allowing tokens without a specific URI to use the collection-level `_uri`:

```solidity
// OpenZeppelin ERC1155URIStorage.sol
function uri(uint256 tokenId) public view virtual override returns (string memory) {
    string memory tokenURI = _tokenURIs[tokenId];
    
    return bytes(tokenURI).length > 0 ? string.concat(_uri, tokenURI) : _uri;
}
```

Trade-off:

* Adding a conditional check to `uri()` increases view-call gas slightly but is a view-only cost.
* Avoiding SSTORE operations in `mint()` (by not storing per-token URIs when unnecessary) reduces expensive write gas on every mint, which is much more impactful.

Real-world example (BelongCheckIn):

* Venue and promoter tokens often share metadata. Both tokens are minted with full URIs, duplicating storage writes.
* Using a collection base URI (e.g., `https://api.belong.com/metadata/venue/`) and omitting per-token URIs lets clients compute token metadata by replacing `id` per ERC-1155 rules, avoiding per-token SSTORE.

### Impact Details

This is classified as an **Insight**: the contract works as intended and is secure, but the current design causes avoidable operational gas costs when many tokens share metadata patterns. As adoption scales, these costs may become significant.

### References

Project contracts:

* contracts/v2/tokens/base/ERC1155Base.sol

Relevant OpenZeppelin references:

* <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC1155/ERC1155.sol>
* <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC1155/extensions/ERC1155URIStorage.sol>

## Recommendation

Implement a fallback pattern aligned with OpenZeppelin's `ERC1155URIStorage` and the ERC-1155 metadata expectations.

Add a fallback to the collection-level URI in `uri()` so that per-token storage writes are optional:

```solidity
// ERC1155Base.sol
function uri(uint256 tokenId) public view override returns (string memory) {
    string memory tokenURI = _tokenUri[tokenId];
    
    return bytes(tokenURI).length > 0 ? string.concat(_uri, tokenURI) : _uri;
}
```

By using this pattern:

* mints can omit the `tokenUri` parameter (or pass an empty string) for tokens that should use the collection base URI,
* the expensive per-token SSTORE is avoided,
* clients still receive correct metadata (base URI + id substitution) per ERC-1155 standard.

{% hint style="info" %}
This is an efficiency recommendation — not a security vulnerability. Apply if reducing mint gas costs is a priority for the project.
{% endhint %}

## Proof of Concept

<details>

<summary>Link to PoC gist</summary>

<https://gist.github.com/SproutGoodHub/ea27370acac22de7272f49b1016b34bc>

</details>

<details>

<summary>Test Setup &#x26; Test Case</summary>

File: `test/v2/tokens/creditToken.test.ts`

```typescript
describe('URI Storage Gas Waste', () => {
  it('Gas consumption comparison: short URI vs long URI', async () => {
    const { venueToken, admin, minter } = await loadFixture(fixture);

    // Mint with short URI (1 character)
    const shortUriMint = await venueToken.connect(minter).mint(admin.address, 1, 1000, '1');
    const shortUriReceipt = await shortUriMint.wait();
    const shortUriGas = shortUriReceipt.gasUsed;

    // Mint with long URI (typical metadata URL - 40 characters)
    const longUri = 'https://api.example.com/metadata/1';
    const longUriMint = await venueToken.connect(minter).mint(admin.address, 2, 1000, longUri);
    const longUriReceipt = await longUriMint.wait();
    const longUriGas = longUriReceipt.gasUsed;

    // Calculate gas difference
    const gasDifference = longUriGas.sub(shortUriGas);
    const percentageIncrease = gasDifference.mul(10000).div(shortUriGas).toNumber() / 100;

    console.log(`Short URI ("1"):        ${shortUriGas.toString()} gas`);
    console.log(`Long URI (40 chars):    ${longUriGas.toString()} gas`);
    console.log(`Gas Difference:         ${gasDifference.toString()} gas (+${percentageIncrease.toFixed(2)}%)\n`);
    
    // Assert significant gas difference
    expect(gasDifference).to.be.gt(35000);
  });
});
```

Run:

```bash
npx hardhat test test/v2/tokens/creditToken.test.ts --grep "Gas consumption"
```

</details>

<details>

<summary>Test Results (console &#x26; reporter)</summary>

Console output:

```
=== URI Storage Gas Consumption ===
Short URI ("1"):        80283 gas
Long URI (40 chars):    125536 gas
Gas Difference:         45253 gas (+56.36%)

      √ Gas consumption comparison: short URI vs long URI
```

Hardhat Gas Reporter excerpt:

```
┌──────────────┬────────────┬─────────┬─────────┬─────────┬─────────────┬────────────┐
│  Contract    │  Method    │  Min    │  Max    │  Avg    │  # calls    │  usd (avg) │
├──────────────┼────────────┼─────────┼─────────┼─────────┼─────────────┼────────────┤
│  CreditToken │  mint      │  80,283 │ 125,536 │ 102,910 │      4      │      -     │
└──────────────┴────────────┴─────────┴─────────┴─────────┴─────────────┴────────────┘
```

</details>
