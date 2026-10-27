# 31226 - \[SC - Insight] Missing Revert Message in require statement lea...

Submitted on May 15th 2024 at 08:49:18 UTC by @Wizard for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31226

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* missing revert message leading to difficulty in debugging

## Description

## Brief/Intro

require statement in the `deposit`, `withdraw`, and `restVoting` functions does not include a revert message.

## Vulnerability Details

The require statement in the `deposit`, `withdraw`, and `restVoting` functions does not include a revert message, this would make it harder to debug and understand why a transaction is reverting for both developers and protocol users.

## Impact Details

the lack of feedback on reverting functions would make it harder to trace back the errors and know where the execution is going wrong, this doesn't have a direct security impact on the contract, but it can make things easier to debug as those three functions are important functions in the protocol.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol?utm\\_source=immunefi#L303>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol?utm\\_source=immunefi#L319>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol?utm\\_source=immunefi#L332>

## Proof of Concept

````
/// @inheritdoc IBribe
    function deposit(uint256 amount, uint256 tokenId) external {
        //@audit missing revert message
      --  require(msg.sender == voter);
      ++  require(msg.sender == voter, " not a voter");
        totalSupply += amount;
        balanceOf[tokenId] += amount;

        totalVoting += amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();
        _writeVotingCheckpoint();

        emit Deposit(msg.sender, tokenId, amount);
    }

    /// @inheritdoc IBribe
    function withdraw(uint256 amount, uint256 tokenId) external {
         -- require(msg.sender == voter);
       ++  require(msg.sender == voter, " not a voter");

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }

    /// @inheritdoc IBribe
    function resetVoting() external {
      require(msg.sender == voter);
  ++  require(msg.sender == voter, " not a voter");

        totalVoting = 0;
    }```
````
