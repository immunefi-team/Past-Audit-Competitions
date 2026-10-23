# 30951 - \[SC - Low] Incorrect ownerOf implementation makes veALCX n...

Submitted on May 8th 2024 at 21:43:15 UTC by @marchev for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30951

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The ERC721 standard requires that ownership queries for NFTs assigned to the zero address must revert, indicating invalid/non-existent NFT. However, the `VotingEscrow#ownerOf()` implementation in veALCX returns `address(0)` instead, violating this standard. This deviation risks causing functional and security issues in external systems that integrate with veALCX and rely on standard ERC721 behavior.

## Vulnerability Details

The [ERC721 specification](https://eips.ethereum.org/EIPS/eip-721) states the following:

```
    /// @notice Find the owner of an NFT
    /// @dev NFTs assigned to zero address are considered invalid, and queries
    ///  about them do throw.
    /// @param _tokenId The identifier for an NFT
    /// @return The address of the owner of the NFT
    function ownerOf(uint256 _tokenId) external view returns (address);
```

This means that every compliant ERC721 token must implement the `ownerOf()` function so that it throws if a `_tokenId` is passed that belongs to `address(0)`. However, `VotingEscrow`'s implementation looks like this:

```sol
    /// @inheritdoc IVotingEscrow
    function ownerOf(uint256 _tokenId) public view override(IERC721, IVotingEscrow) returns (address) {
        return idToOwner[_tokenId];
    }
```

For a non-existent `_tokenId`, this implementation will return `address(0)` instead of reverting. This behavior is not compliant with the ERC721 specification.

This non-compliance can lead to serious compatibility issues with external integrations that rely on standard ERC721 behavior. Applications expecting a revert when querying ownership of an invalid NFT might instead receive a non-reverting response, potentially resulting in erroneous processing or security vulnerabilities in systems interacting with veALCX.

## Impact Details

Non-compliance with the ERC721 standard in veALCX's implementation could lead to interoperability issues and security vulnerabilities in external systems that rely on standard behavior for processing ownership data. This could result in erroneous executions and potential breaches in decentralized applications interfacing with veALCX.

To better demonstrate the potential impact of this vulnerability, I have developed a PoC that showcases how this issue could lead to a permanent loss of NFTs. Please see it in the Proof of Concept section below.

The designation of the vulnerability as **Medium** severity is justified given its high potential impact on the disruption of integrations with other protocols. Similar vulnerabilities have been reported and classified with medium severity, underscoring the consistency in assessing the potential risks associated with non-compliance to the ERC721 standard. Examples:

<https://solodit.xyz/issues/m-19-holographerc721approve-not-eip-721-compliant-code4rena-holograph-holograph-contest-git>

<https://solodit.xyz/issues/m-02-violation-of-erc-721-standard-in-verbstokentokenuri-implementation-code4rena-collective-collective-git>

<https://solodit.xyz/issues/m-24-mintableincentivizederc721-and-ntoken-do-not-comply-with-erc721-breaking-composability-code4rena-paraspace-paraspace-contest-git>

## References

Problematic implementation:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L230-L232>

Docs references which state that ERC721 compliance is expected:

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L16>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/CONTRACTS.md>

## Proof of Concept

Let's examine a hypothetical scenario:

**Context:** LiquidityHub is a fictional platform where users can stake their veALCX. In return, they receive enhanced rewards. The LiquidityHub manages these veALCXes, allowing users to stake on behalf of the hub and later redeem them. Upon redemption, LiquidityHub deducts an exit service fee before returning the remaining balance to the user.

Here’s how the vulnerability could result in Alice's veALCX becoming irretrievably stuck:

1. Alice uses the `VotingEscrow` contract to create a veALCX lock on behalf of the `LiquidityHub`, which assigns the ownership of the veALCX to the LiquidityHub.
2. Alice calls the `stake()` function on the `LiquidityHub` contract with the veNFT's token ID. The function verifies that the LiquidityHub is the owner before marking it as staked.
3. When Alice wishes to redeem, she calls `startCooldown()` and after the cooldown period expires triggers the `redeem()` function in the LiquidityHub.
4. The LiquidityHub calls `withdraw()` in the `VotingEscrow` to "burn" the veALCX and withdraw the BPT tokens.
5. The LiquidityHub checks the burn status by expecting the `ownerOf()` call to revert, indicating the token no longer exists.
6. If `ownerOf()` does not revert and returns any address, the LiquidityHub reverts the transaction, signaling an unsuccessful burn. If `ownerOf()` reverts, the LiquidityHub clears its record of the token, recognizing the successful redemption and burn.

The following coded PoC code illustrates the potential impact and how the veALCX can remain stuck due to a failed validation of its burned status.

Add the following contract at the end of `VotingEscrow.t.sol`:

```sol

contract LiquidityHub {
    VotingEscrow public veALCX;
    mapping(uint256 => address) public stakedTokens;

    constructor(address _veALCX) {
        veALCX = VotingEscrow(_veALCX);
    }

    // Function for Alice to stake her veALCX which is already in the name of this hub
    function stake(uint256 tokenId) external {
        require(veALCX.ownerOf(tokenId) == address(this), "Hub is not the owner");

        // Marking the token as staked
        stakedTokens[tokenId] = msg.sender;
    }

    function startCooldown(uint256 tokenId) external {
        require(veALCX.ownerOf(tokenId) == address(this), "Hub is not the owner");
        require(stakedTokens[tokenId] == msg.sender, "Not staked by you");

        veALCX.startCooldown(tokenId);
    }

    // Function to redeem the veALCX and encounter the issue
    function redeem(uint256 tokenId) external {
        require(stakedTokens[tokenId] == msg.sender, "Not staked by you");

        // Call withdraw from veALCX
        veALCX.withdraw(tokenId);

        // Use try-catch to check if the token has been successfully burned
        try veALCX.ownerOf(tokenId) {
            // If this call does not fail, then assume the token was not burned successfully
            revert("Token not burned properly");
        } catch {
            // Catch the error to confirm token is burned
            // Only reach here if ownerOf() reverts, which is expected upon successful burn
        }

        // Clear the staking record
        delete stakedTokens[tokenId];

        // Additional logic to handle the release of funds or other benefits to the user
        // E.g. send 5% fee to LiquidityHub treasury + remaining funds to Alice
    }
}
```

Also, add the following test case to `VotingEscrowTest`:

```sol
    function test_incorrect_erc721_ownerof_implementation_can_lead_to_nft_loss() public {
        address alice = address(31337);
        LiquidityHub liquidityHub = new LiquidityHub(address(veALCX));
        deal(bpt, alice, 50 ether); // Make sure Alice has some BPT

        vm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), 10 ether);
        uint256 tokenId = veALCX.createLockFor(10 ether, 180 days, false, address(liquidityHub));
        liquidityHub.stake(tokenId);

        vm.warp(block.timestamp + 181 days);

        liquidityHub.startCooldown(tokenId);

        vm.warp(block.timestamp + 7 days);

        liquidityHub.redeem(tokenId);
        vm.stopPrank();
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_incorrect_erc721_ownerof_implementation_can_lead_to_nft_loss
```

Run the PoC via `make test_file_test`

This example is streamlined to focus on the core issue, leaving out many details of the LiquidityHub’s operations for clarity.
