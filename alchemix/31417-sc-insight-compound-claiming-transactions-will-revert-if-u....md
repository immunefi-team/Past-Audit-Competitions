# 31417 - \[SC - Insight] Compound claiming transactions will revert if u...

Submitted on May 18th 2024 at 21:09:37 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31417

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

When user tries to reinvest their claimed reward with `_compound = true` in `RewardsDistributor.claim(uint256 _tokenId, bool _compound)`, it will always revert when users `_tokenId` is expired.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L174-L189>

```solidity
if (_compound) {
	(uint256 wethAmount, uint256[] memory normalizedWeights) = amountToCompound(alcxAmount);

	require(
		msg.value >= wethAmount || WETH.balanceOf(msg.sender) >= wethAmount,
		"insufficient balance to compound"
	);

	// Wrap eth if necessary
	if (msg.value > 0) {
		WETH.deposit{ value: wethAmount }();
	} else IERC20(address(WETH)).safeTransferFrom(msg.sender, address(this), wethAmount);

	_depositIntoBalancerPool(wethAmount, alcxAmount, normalizedWeights);

	IVotingEscrow(votingEscrow).depositFor(_tokenId, IERC20(lockedToken).balanceOf(address(this))); //@audit
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L667-L672>

```solidity
function depositFor(uint256 _tokenId, uint256 _value) external nonreentrant {
	LockedBalance memory _locked = locked[_tokenId];

	require(_value > 0); // dev: need non-zero value
	require(_locked.amount > 0, "No existing lock found");
	require(_locked.end > block.timestamp, "Cannot add to expired lock. Withdraw"); //@audit
```

As we can see, while performing `depositFor()` from `claim()`, its checking for `require(_locked.end > block.timestamp, "Cannot add to expired lock. Withdraw");`. Which will revert user's claim tx with compounding when their veALCX token is expired.

## Impact

Users compounding claim tx will always revert when their veALCX NFT token is expired. User should not allowed to call claim function with compounding once their veALCX token is expired.

## Recommendation

Consider sending the claimed veALCX rewards to the owner of the veALCX if the veALCX's lock has already expired.

When executing code block of compounding, it should also check along with compounding check that whether veALCX NFT token is not expired as shown below.

```diff
function claim(uint256 _tokenId, bool _compound) external payable nonReentrant returns (uint256) {
    // ...SNIP...

+   uint256 lockEnd = IVotingEscrow(votingEscrow).lockEnd(_tokenId);
+   if (_compound && lockEnd > block.timestamp) {
-   if (_compound) {
        // ...SNIP...
        IVotingEscrow(votingEscrow).depositFor(_tokenId, IERC20(lockedToken).balanceOf(address(this)));

        // ...SNIP...
    } else {
        // ...SNIP...
    }
}
```

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L174-L189>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L667-L672>

## Proof Of Concept

**Note:** For PoC to work, we need to move time beyond 60 days which is hardcoded in `RewardsDistributor` contract as `staleThreshold = 60 days` at <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L118>. Since time has moved beyond 60 days but oracle(priceFeed) has not updated during this time interval, its reverting with `Price stale` message. To mimic oracle update, we have updated `updatedAt` field returns from `latestRoundData()` to avoid `Price stale` revert using foundry cheatcode - `load` and `store`.

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/Minter.t.sol`
* Run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract MinterTest --match-test testCompoundRewardFailureOnVeALCXExpired -vv`

```solidity
// Compound claiming should revert if users veALCX is expired
function testCompoundRewardFailureOnVeALCXExpired() public {
    initializeVotingEscrow();

    hevm.startPrank(admin);

    // After no epoch has passed, amount claimable should be 0
    assertEq(distributor.claimable(tokenId), 0, "amount claimable should be 0");

    // Fast forward to new epoch
    hevm.warp(newEpoch());
    hevm.roll(block.number + 1);

    voter.distribute();

    // Accounts must provide proportional amount of WETH to deposit into the Balancer pool
    weth.approve(address(distributor), type(uint256).max);

    // Fast forward to MAXTIME to expire tokenId
    uint expiry = veALCX.lockEnd(tokenId);
    hevm.warp(expiry + 1 seconds);
    hevm.roll(block.number + 1);

    // Update priceFeed's `updatedAt` field
    address aggregator = 0x64a119DCf78E7E3FcED89c429f6F47Bf0cd80250;
    uint transmissionSlot = 44; //https://evm.storage/eth/19889196/0x64a119dcf78e7e3fced89c429f6f47bf0cd80250/s_transmissions#map
    uint latestRoundId = 2608;
    bytes32 loc = keccak256(abi.encode(latestRoundId, transmissionSlot));
    bytes32 data = hevm.load(aggregator, loc);
    data = data & 0x0000000000000000ffffffffffffffffffffffffffffffffffffffffffffffff;
    data = data | bytes32(block.timestamp << 192);
    hevm.store(aggregator, loc, data);

    // Claim with compunding should revert
    hevm.expectRevert("Cannot add to expired lock. Withdraw");
    distributor.claim(tokenId, true);

    hevm.stopPrank();
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract MinterTest --match-test testCompoundRewardFailureOnVeALCXExpired -vv

[⠒] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/Minter.t.sol:MinterTest
[PASS] testCompoundRewardFailureOnVeALCXExpired() (gas: 7409861)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.73ms (4.62ms CPU time)
```
