# 31460 - \[SC - Insight] supportsInterface does not return typeIERCRecei...

Submitted on May 19th 2024 at 21:40:50 UTC by @OxRizwan for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31460

Report type: Smart Contract

Report severity: Insight

Target: <https://immunefi.com>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`supportsInterface()` does not return `type(IERC721Receiver).interfaceId`

## Vulnerability Details

`AlchemixGovernor.sol` contract has inherited `L2Governor` abstract as base contract. This base contract has `supportsInterface()` function which returns the supported interface id by Alchemix Governor contract. It should be noted that Alchemix governor contract supports both `IERC721Receiver` interface id and `IERC1155Receiver` interface id, however, the implementation of `supportsInterface` has only returned `IERC1155Receiver` and does not return the `IERC721Receiver` interface id and this can be seen as below:

```solidity
    function supportsInterface(bytes4 interfaceId) public view virtual override(IERC165, ERC165) returns (bool) {
        // In addition to the current interfaceId, also support previous version of the interfaceId that did not
        // include the castVoteWithReasonAndParams() function as standard
        return
            interfaceId ==
            (type(IGovernor).interfaceId ^
                this.castVoteWithReasonAndParams.selector ^
                this.castVoteWithReasonAndParamsBySig.selector ^
                this.getVotesWithParams.selector) ||
            interfaceId == type(IGovernor).interfaceId ||
            interfaceId == type(IERC1155Receiver).interfaceId ||
            super.supportsInterface(interfaceId);
    }
```

## Impact Details

`supportsInterface()` does not explicitely return the `interfaceId == type(IERC721Receiver).interfaceId` while calling it. The Alchemix governor supports the ERC721Receiver but the interface id does not return it so its a low severity issue

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2Governor.sol#L130-L142>

## Recommendation to fix

```diff
    function supportsInterface(bytes4 interfaceId) public view virtual override(IERC165, ERC165) returns (bool) {
        // In addition to the current interfaceId, also support previous version of the interfaceId that did not
        // include the castVoteWithReasonAndParams() function as standard
        return
            interfaceId ==
            (type(IGovernor).interfaceId ^
                this.castVoteWithReasonAndParams.selector ^
                this.castVoteWithReasonAndParamsBySig.selector ^
                this.getVotesWithParams.selector) ||
            interfaceId == type(IGovernor).interfaceId ||
+          interfaceId == type(IERC721Receiver).interfaceId ||
            interfaceId == type(IERC1155Receiver).interfaceId ||
            super.supportsInterface(interfaceId);
    }
```

## Proof of Concept

The issue is simple to understand and very straight forward. It explains the interfaceId function unable to return the promised return i.e `interfaceId == type(IERC721Receiver).interfaceId` as true.

The issue is not complex so i believe it can be easily understood with above description and recommendation.

Thank you for your understanding.
