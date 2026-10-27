# 30992 - \[SC - Insight] Inconsistent State Missing Event Emission in Fl...

Submitted on May 10th 2024 at 08:29:50 UTC by @Wizard for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30992

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* contracts or users may not be aware that an NFT has been claimed, leading to inconsistent state

## Description

## Brief/Intro

The FluxToken.sol::nftClaim function does not emit an event when an NFT is claimed, which can lead to inconsistent state.

## Vulnerability Details

The nftClaim function marks an NFT as claimed by setting claimed\[\_nft]\[\_tokenId] = true, but it does not emit an event to notify external contracts that the NFT has been claimed. This is an important step in tracking claimed NFTs.

## Impact Details

While there are no direct financial losses associated with the missing event., not emitting an event would make logging and tracing claimed NFTs much harder for external contracts relying on the function's input.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol?utm\\_source=immunefi#L134>

## Proof of Concept

```
event ClaimedNFT(address indexed _nft, uint256 indexed _tokenId, address indexed _claimer);

                            --------Code--------- 

 function nftClaim(address _nft, uint256 _tokenId) external {
        // require claim to be within a year of deploy date
        require(block.timestamp < deployDate + oneYear, "claim period has passed");

        require(!claimed[_nft][_tokenId], "already claimed");

        // value of the NFT
        uint256 tokenData = 0;

        // determine which nft is being claimed
        if (_nft == alchemechNFT) {
            // require sender to be owner of the NFT
            require(IAlchemechNFT(_nft).ownerOf(_tokenId) == msg.sender, "not owner of Alchemech NFT");

            tokenData = IAlchemechNFT(_nft).tokenData(_tokenId);
        } else if (_nft == patronNFT) {
            // require sender to be owner of the NFT
            require(IAlEthNFT(_nft).ownerOf(_tokenId) == msg.sender, "not owner of Patron NFT");

            tokenData = IAlEthNFT(_nft).tokenData(_tokenId);
        } else {
            revert("invalid NFT");
        }

        // mark the token as claimed
        claimed[_nft][_tokenId] = true;
++      emit ClaimedNFT(_nft, _tokenId, msg.sender);

        uint256 amount = getClaimableFlux(tokenData, _nft);

        _mint(msg.sender, amount);
    }

```
