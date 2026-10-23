# 31592 - \[SC - Insight] Collection of other important issues

Submitted on May 21st 2024 at 15:27:42 UTC by @Minato7namikazi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31592

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Collection of other important issues

## Important

**take a look first at the attachments .. i put the same first 4 issues but in more structured way**

## 1. require(treasuryPct <= BPS) implemented wrong in the constructor

In the constructor of `RevenueHandler`, there is a logical check for the treasury percentage (`treasuryPct`) that is intended to ensure it does not exceed a certain threshold (100%, or `BPS` which is 10,000 basis points) ,there is a logical error in the order of operations for setting the `treasuryPct`. Here's the specific constructor code:

```
   constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {

       veALCX = _veALCX;
       require(_treasury != address(0), "treasury cannot be 0x0");
       treasury = _treasury;
       require(treasuryPct <= BPS, "treasury pct too large");
       treasuryPct = _treasuryPct;

   }
```

The check `require(treasuryPct <= BPS, "treasury pct too large");` is performed before `treasuryPct` is actually set to `_treasuryPct`. At the point of the check, `treasuryPct` is still its default value (which is 0 since it's a state variable). Therefore, **this check is essentially redundant and does not validate `_treasuryPct`.** - The correct value `_treasuryPct` is only assigned to `treasuryPct` after the check, meaning any value could be set

<------------------------------------------------------------------->

## 2. in poke function it doesn't check if the token is expired

the only way to call internal \_vote() function in the voter contract is via either of vote() or poke()

vote() implement a check that the token is not expired in this part before it call the internal \_vote() function

```
        require(block.timestamp < IVotingEscrow(veALCX).lockEnd(_tokenId), "cannot vote with expired token");
        _vote(_tokenId, _poolVote, _weights, _boost);
```

#### but poke() doesn't!

<------------------------------------------------------------------->

## 3. missed check of isgauge alive in \_updatefor in voter.sol

in the function

```
/**

_updateFor function is responsible for updating the supply index and accruing the claimable rewards for a specific gauge address.
It does this by calculating the difference between the current global index and the previous supply index for the gauge,
and then adding the accrued share to the claimable mapping.

 */
    function _updateFor(address _gauge) internal {

        require(isGauge[_gauge], "invalid gauge");

        address _pool = poolForGauge[_gauge];
        uint256 _supplied = weights[_pool];

        if (_supplied > 0) {
            uint256 _supplyIndex = supplyIndex[_gauge];
            uint256 _index = index; // get global index0 for accumulated distro
            supplyIndex[_gauge] = _index; // update _gauge current position to global position
            uint256 _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued

            if (_delta > 0) {

                uint256 _share = (uint256(_supplied) * _delta) / 1e18; // add accrued difference for each supplied token
                claimable[_gauge] += _share;
            }
        } else {
            supplyIndex[_gauge] = index;
        }
    }

```

When the killGauge is invoked, the claimable gauge distributions are frozen. This means that when a gauge is terminated, the value associated with the claimable\[\_gauge] key is cleared. This is because any rewards received by the Voter contract are indexed and distributed proportionally to the weight of each pool. As a result, the claimable amount becomes permanently locked within the contract and can no longer be accessed or claimed.

So the **Recommendation**: Instead of permanently locking the claimable amount within the contract, consider returning it to the Minter contract. This would ensure that the funds are not lost or inaccessible. Additionally, it may be beneficial to wipe the existing votes on the killed gauge. This is because the killed gauge will continue to accumulate rewards from the Minter contract, even though it is no longer active. Wiping the votes would prevent this from happening and ensure a clean slate after the gauge is terminated

so edit the function before it apply `claimable[_gauge] += _share;`

to check first if the gauge is alive .... the updated function part should be

```
 if (isAlive[_gauge]) {
                    claimable[_gauge] += _share;
                } else {
                    IERC20(rewardToken).safeTransfer(minter, _share); /
                }
```

<------------------------------------------------------------------->

## 4. Inconsistent between balanceOfToken() and balanceOfTokenAt() functions

the balanceOfToken() function implements a flash-loan protection that returns zero voting balance if `ownershipChange[_tokenId] == block.number` ,

```
    function balanceOfToken(uint256 _tokenId) external view returns (uint256) {
        if (ownershipChange[_tokenId] == block.number) return 0;
        return _balanceOfTokenAt(_tokenId, block.timestamp);
    }
```

this was not consistently applied to the balanceOfTokenAt()

```
    function balanceOfTokenAt(uint256 _tokenId, uint256 _time) external view returns (uint256) {
        return _balanceOfTokenAt(_tokenId, _time);
    }
```

As a consequence, when external protocols or alchemix call the balanceOfToken and balanceOfTokenAt functions, they will receive different voting balances for the same veALCXs, depending on which function they use. Additionally, the internal \_balanceOfTokenAt function, which lacks flash-loan protection, is called by the VotingEscrow\.getVotes function to calculate the voting balance of an account. While the VotingEscrow\.getVotes function does not appear to be used in any in-scope contracts, it might be utilized by external protocols or off-chain components to tally the votes. If that is the case, a malicious user could exploit this by flash-loaning the veALCXs to artificially inflate their voting balance.

Recommendation: If the requirement is to have all newly transferred veALCXs (where ownershipChange\[\_tokenId] == block.number) have a zero voting balance to prevent someone from flash-loaning veALCXs to increase their voting power, the flash-loan protection should be consistently implemented across all the related functions.

<------------------------------------------------------------------->

## 5. All rewards will be lost until Gauge or Bribe deposits are non-zero

Because the rewards are emitted over DURATION, if no deposit has happened and notifyRewardAmount() is called\
with a non-zero value, all rewards will be forfeited until totalSupply is non-zero as nobody will be able to claim\
them.

Recommendation: Document this risk to end users and tell them to deposit before voting on a gauge.

<------------------------------------------------------------------->

## 6. Difference in getPastTotalSupply and propose

in alchemix governor

```
    function proposalThreshold() public view override(L2Governor) returns (uint256) {
        return (token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR;
    }
```

The getPastTotalSupply() function currently uses block.timestamp, but OpenZeppelin's propose()\
function will use votes from block.number - 1 as seen here

```
    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) public virtual override returns (uint256) {

        require(
            getVotes(_msgSender(), block.number - 1) >= proposalThreshold(),
            "Governor: proposer votes below proposal threshold"
        );

```

This could enable

• Front-run and increase total supply to cause proposer to be unable to propose().\
• Require higher tokens than expected if total supply can grow within one block.\
Proposals could be denied as long as a whale is willing to lock more tokens to increase the total supply and thereby\
increase the proposal threshold.\
Recommendation: Consider computing total supply and votes values in the same block.

<------------------------------------------------------------------->

## 7. RewardDistributor caching totalSupply leading to incorrect reward calculation

RewardDistributor distributes newly minted ALCX tokens to users who locks the tokens in VotingEscrow. Since the calculation of past supply is costly, the rewardDistributor cache the supply value in\
uint256\[1000000000000000] public veSupply. The RewardDistributor.\_checkpointTotalSupply function\
would iterate from the last updated time util the latest epoch time, fetches totalSupply from votingEscrow, and store it.\
Assume the following scenario when a transaction is executed at the beginning of an epoch.

1. The totalSupply is X.
2. The user calls checkpointTotalSupply. The rewardDistributor save the totalSupply = X.
3. The user creates a lock with 2X the amount of tokens. The user has balance = 2X and the totalSupply becomes 3X.
4. Fast forward to when the reward is distributed. The user claims the tokens, reward is calculated by total reward \* balance / supply and user gets 2x of the total rewards.

Recommendation: The quick fix would be to stop rewardDistributor caching totalSupply when it can still increase.

```
function _checkpointTotalSupply() internal {
address _ve = ve;
uint256 t = timeCursor;
uint256 roundedTimestamp = (block.timestamp / WEEK) * WEEK;
IVotingEscrow(_ve).checkpoint();
for (uint256 i = 0; i < 20; i++) {

// Mitigation : Replace  ">"  with  ">=" 

- if (t > roundedTimestamp) {
+ if (t >= roundedTimestamp) {


break;
} else {
// fetch last global checkpoint prior to time t
uint256 epoch = _findTimestampEpoch(t);
IVotingEscrow.GlobalPoint memory pt = IVotingEscrow(_ve).pointHistory(epoch);
int128 dt = 0;
if (t > pt.ts) {
dt = int128(int256(t - pt.ts));
}
// walk forward voting power to time t
veSupply[t] = uint256(int256(max(pt.bias - pt.slope * dt, 0))) +
pt.permanentLockBalance;,!
}
t += WEEK;
}
timeCursor = t;
}
```

## 8. `Ownable` uses single-step ownership transfer

Likelihood: Low, because it requires an error on the admin side

Impact: High, because important protocol functionality will be bricked

the ownable in revenueHandler contract

the commonly used Openzeppelin ownable implementation has a shortcoming that it allows the owner to transfer ownership to a non-existent or mistyped address.

It is a best practice to use Ownable2Step because it's safer than Ownable for smart contracts because the owner cannot accidentally transfer smart contract ownership to a mistyped address. Rather than directly transferring to the new owner, the transfer only completes when the new owner accepts ownership.

So it's important to consider using the **OpenZeppelin's** new Ownable2Step contract

## Proof of Concept
