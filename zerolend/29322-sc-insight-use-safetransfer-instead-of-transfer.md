# 29322 - \[SC - Insight] Use safeTransfer instead of transfer

Submitted on Mar 14th 2024 at 00:40:23 UTC by @bugtester for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29322

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

transfer() might return false instead of reverting, in this case, ignoring return value leads to considering it successful.

use safeTransfer() or check the return value if length of returned data is > 0.

## Vulnerability Details

A call to transferFrom or transfer is frequently done without checking the results. For certain ERC20 tokens, if insufficient tokens are present, no revert occurs but a result of "false" is returned. So its important to check this. in case, ignoring return value leads to considering it successful. and loss of funds

## Impact Details

transfer() might return false instead of reverting, in this case, ignoring return value leads to considering it successful. and cause permanent loss of funds

## Fix

Use safeTransfer instead of transfer

## Proof of concept

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/vesting/VestedZeroNFT.sol#L223C2-L228C1>

function claimUnvested(uint256 tokenId) external { require(msg.sender == stakingBonus, "!stakingBonus"); uint256 \_pending = unclaimed(tokenId); zero.transfer(msg.sender, \_pending); }

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/vesting/VestedZeroNFT.sol#L176C1-L177C1>
