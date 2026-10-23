# 58127 sc critical users can invoke the poke function whenever the lastearmarkdebtblock is exactly one block behind the current block number which lead to affecting users earmarked debt

**Submitted on Oct 30th 2025 at 20:00:30 UTC by @zeroK for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58127
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Theft of unclaimed yield
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The earmarking system updates the earmark weight only when a specific token amount per block is reached. This is achieved by invoking the `queryGraph` function, which in turn calls the queryStake function in stakingGraph.sol. the staking graph library plays a critical role in retrieving and updating the current on-chain stake state. it implements a double Fenwick tree (Binary Indexed Tree) structure for improved efficiency. his process determines the tokens distributed per block.The team has already provided a comprehensive explanation of this mechanism, including how token emissions and redemptions per block are calculated, and how the earmark and redemption processes are interconnected. full details are available in the link below :

<https://docs.google.com/document/d/1tY5\\_v4x6Njbm\\_oBsN3syptDPOW6ptPGX6OgFWrT\\_Xkk/edit?tab=t.0#heading=h.xcg1e2ldkd3a>

since the tokens returned by queryGraph are tokens per block, there is a flow in the earmark function, that can lead to ignore tokens for unlimited number of blocks which lead to prevent accurate update to the earmark weight which affects in returns users required earmark amount will and same the collateral to be locked, this is possible by a simple way, the earmark function invokes call to the queryGraph by setting the block.number as end block and last updated block +1 as start, between these two blocks there is possibility of existing amount that need to be used to update earmarkWeight, but mailcious user can invoke poke() function for each 1 block difference, for example last updated block = 1000 and current block = 1001, returned amount by query should be 0.009 but since \_earmark sets the start as last updated plus one, the queryGraph returns zero due to line below:

```
if (endBlock <= startBlock) return 0;
```

this lead to earmarkWeight to be less accurate, which affect users earmarked debt and the survival mechanism too.

## Vulnerability Details

the function \_earmark implemented as shown below:

```solidity
function _earmark() internal {
        if (totalDebt == 0) return;
        if (block.number <= lastEarmarkBlock) return;

        // Yield the transmuter accumulated since last earmark (cover)
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

        // Proper saturating subtract in DEBT units
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;

        lastTransmuterTokenBalance = transmuterCurrentBalance;

        uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
        if (amount > liveUnearmarked) amount = liveUnearmarked;

        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
            _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

            cumulativeEarmarked += amount;
        }

        lastEarmarkBlock = block.number;
    }

```

there is some line that is important to focus on to understand this report, first thing we can see the first check `if (block.number <= lastEarmarkBlock)` which prevents updating in same block, while its critical check but it won't prevent the malicious behavior since lastEarmarkBlock become block.number + 1 later when queryGraph invoked. the second thing is how lastEarmarkBlock get updated even if the earmarkWeight/\_survivalAccumulator not updated, this give the malicious user to execute its attack, but lets first take a look at query graph function too:

```solidity
function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {
        if (endBlock <= startBlock) return 0;

        int256 queried = _stakingGraph.queryStake(startBlock, endBlock);
        if (queried == 0) return 0;

        // You currently add +1 for rounding; keep in mind this can create off-by-one deltas.
        return (queried / BLOCK_SCALING_FACTOR).toUint256()
            + (queried % BLOCK_SCALING_FACTOR == 0 ? 0 : 1);
    }

```

all we need in this line is the check `if (endBlock <= startBlock) return 0;` which will be executed in our case, now the malicious user can invoke the steps below to make lose of tokens per one block possible.

* alice invokes the poke() function, it invokes \_earmark and \_sync without any modifier or requirement. this will update cumulative earmark, earmarkWeight and \_survivalAccumulator, and more important, it sets lastEarmarkedBlock to curren block.number(in our case block 1000 for simplicity).
* after block.number become 1001, alice invoke poke() again, which leads the line \`if (endBlock <= startBlock) and return zero, while it should return tokens per block and update earmarkWeight and other important state.
* the amount returned in the line `ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);` equal to zero this is because `lastEarmarkBlock + 1 = 1001, block.number = 1001`which lead to queryGraph to return zero early.
* the lastEarmarkBlock will get updated to current block.number, which open the possibility of repeating this attack.
* this way the earmarkweight and other important state will not be updated each time alice invokes poke() when difference between lastEarmarkBlock and block.number is one block only, which affect users earmarked debt and the whole ecosystem mechanism.
* while tokens per one block can be between 1-100 tokens depends on redemption demand, but repeating this behavior for up to +10 times will lead to consequences, and there is no modifier or safety action that prevent attacker to invoke this action except for quick update to the core implementation which can take at least 30 min(more than enough for attacker to cause real damage)

## Impact Details

attacker can affect earmark mechanism by invoke call to \_earmark function through poke() function for each one block differences.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1098C5-L1132C6>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L654-L658>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L270>

## Proof of Concept

## Proof of Concept

run the test below in `transmuter.t.sol`:

```solidity

    function testDiffOneBlockLose() public {
        vm.prank(address(0xbeef));
        transmuter.createRedemption(100e18);
        Transmuter.StakingPosition memory position = transmuter.getPosition(1);
        //user 2
        vm.startPrank(address(0xdad));
        alETH.approve(address(transmuter), type(uint256).max);
        transmuter.createRedemption(100e18);
        Transmuter.StakingPosition memory position1 = transmuter.getPosition(1);
        // user 3
        vm.startPrank(address(0x1));
        alETH.approve(address(transmuter), type(uint256).max);
        transmuter.createRedemption(100e18);
        Transmuter.StakingPosition memory position2 = transmuter.getPosition(1);
        //user 3
        vm.startPrank(address(0x2));
        alETH.approve(address(transmuter), type(uint256).max);
        transmuter.createRedemption(100e18);
        Transmuter.StakingPosition memory position3 = transmuter.getPosition(1);

        uint blockTimePlus = block.number + 1;
        uint blockTime = block.number ;
        vm.roll(block.number + 1);
        //SHOW how much it should return per blow that get lost due to the incorrect flow
        uint val = transmuter.queryGraph(blockTime, block.number);
        console.log("Graph query value that will get lost:", val);
        uint val_lost = transmuter.queryGraph(blockTimePlus, block.number);
        console.log("Graph query value, returns zero since start == end :", val_lost);

    }


```
