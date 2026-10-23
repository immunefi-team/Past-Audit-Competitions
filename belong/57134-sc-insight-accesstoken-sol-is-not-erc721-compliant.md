# 57134 sc insight accesstoken sol is not erc721 compliant

Submitted on Oct 23rd 2025 at 19:18:51 UTC by @kaysoft for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57134
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* Impacts:
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

AccessToken.sol is an ERC721 contract but its `supportsInterface(...)` function returns `false` for the ERC721 interface ID.

When NFT marketplaces, wallets and NFT bridges try to interact with AccessToken, they call `supportsInterface(...)` to detect implemented interfaces. Currently `supportsInterface(...)` returns false for the ERC721 interface ID (0x80ac58cd).

### Vulnerability Details

According to ERC-721 specification:

> Every ERC-721 compliant contract must implement the ERC721 and ERC165 interfaces

The `supportsInterface(...)` function in AccessToken.sol currently returns `false` for the ERC721 interface ID (0x80ac58cd). This is caused by incorrect usage of `super.supportsInterface(...)` combined with Solidity inheritance linearization — the `super` call resolves to the implementation of the most derived parent according to C3 linearization. In the current contract layout, `super.supportsInterface(...)` resolves to the `ERC2981` implementation, so the `ERC721` parent implementation is not checked.

Current implementation snippet:

```solidity
function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC2981) returns (bool) {
        bool result;
        /// @solidity memory-safe-assembly
        assembly {
            let s := shr(224, interfaceId)
            // ICreatorToken: 0xad0d7f6c, ILegacyCreatorToken: 0xa07d229a.
            // ERC4906: 0x49064906, check https://eips.ethereum.org/EIPS/eip-4906.
            result := or(or(eq(s, 0xad0d7f6c), eq(s, 0xa07d229a)), eq(s, 0x49064906))
        }

        return result || super.supportsInterface(interfaceId);//@audit super only ref ERC2981 not ERC721
    }
```

### Impact Details

* AccessToken may not integrate with existing NFT marketplaces and wallets (they rely on ERC165 `supportsInterface` to detect ERC721 support).

## Recommendation

Replace the single `super.supportsInterface(interfaceId)` call with explicit checks for both parent implementations so both ERC2981 and ERC721 interface checks are performed. Example patch:

```diff
function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC2981) returns (bool) {
        bool result;
        /// @solidity memory-safe-assembly
        assembly {
            let s := shr(224, interfaceId)
            // ICreatorToken: 0xad0d7f6c, ILegacyCreatorToken: 0xa07d229a.
            // ERC4906: 0x49064906, check https://eips.ethereum.org/EIPS/eip-4906.
            result := or(or(eq(s, 0xad0d7f6c), eq(s, 0xa07d229a)), eq(s, 0x49064906))
        }

--        return result || super.supportsInterface(interfaceId);
++        return result || ERC2981.supportsInterface(interfaceId) || ERC721.supportsInterface(interfaceId);
    }
```

This ensures both parents' `supportsInterface` implementations are considered.

## Proof of Concept

{% stepper %}
{% step %}

### Prepare test

Copy and paste the test below into `accessToken.test.ts` file in the `'Deployment'` test suite.
{% endstep %}

{% step %}

### Run test

Run the test suite:

yarn test
{% endstep %}

{% step %}

### Expected failing test

The test demonstrates that `AccessToken.sol#supportsInterface(...)` returns `false` for both ERC721 and ERC721Metadata interface IDs.

```solidity
it('Should support erc721 and erc721metadata interface ids', async () => {

      const {        
        accessTokenEth,
      } = await loadFixture(fixture);

      //ERC721: 0x80ac58cd, ERC721Metadata: 0x5b5e139f
      const interfaceIdIERC721 = '0x80ac58cd'; // ERC721: 0x80ac58cd
      const interfaceIdIERC721Metadata = '0x5b5e139f'; // ERC721Metadata: 0x5b5e139f
      
      // Returns false for both  ERC721: 0x80ac58cd and ERC721Metadata: 0x5b5e139f
      expect(await accessTokenEth.supportsInterface(interfaceIdIERC721)).to.be.false;
      expect(await accessTokenEth.supportsInterface(interfaceIdIERC721Metadata)).to.be.false;
      
    });
```

{% endstep %}
{% endstepper %}
