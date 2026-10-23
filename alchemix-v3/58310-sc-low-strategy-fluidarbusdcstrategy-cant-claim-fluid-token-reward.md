# 58310 sc low strategy fluidarbusdcstrategy cant claim fluid token reward

**Submitted on Nov 1st 2025 at 06:35:40 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58310
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/FluidARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

Fluid protocol itself provide a reward system that is rewarding its user each cycle specified. The issue is that the `FluidARBUSDCStrategy` is not overriding the `MYTStrategy` reward claim function, making the strategy unable to claim this provided rewards from Fluid protocol.

## Vulnerability Details

`MYTStrategy` already provided `claimRewards` function, but it is not overrided inside `FluidARBUSDCStrategy` . the function `claimRewards` can be called but would do nothing. the details can be seen in the PoC section.

## Impact Details

potential protocol and user losses because the reward cannot be claimed.

## References

<https://arbiscan.io/tx/0x987f844560da00da506d88ee0f4d9de2364c907f64d4b2a07426f8e3bd95d532>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/FluidARBUSDCStrategy.sol#L16-L55>

## Proof of Concept

## Proof of Concept

the setup needs merkle proof, that we generate using this file:

```jsx
// Ethers v6 imports: Get AbiCoder and keccak256 directly
import { AbiCoder, keccak256 } from "ethers";
import { MerkleTree } from "merkletreejs";

// Helper function to create the leaf (updated for v6)
function createLeaf(data) {
  // 1. abi.encode() the data
  // USE AbiCoder.default.encode INSTEAD OF ethers.utils.defaultAbiCoder.encode
  const innerData = AbiCoder.defaultAbiCoder().encode(
    ["uint8", "bytes32", "address", "uint256", "uint256", "bytes"],
    [
      data.positionType,
      data.positionId,
      data.recipient,
      data.cycle,
      data.cumulativeAmount,
      data.metadata
    ]
  );

  // 2. keccak256(inner_data)
  // USE keccak256() DIRECTLY INSTEAD OF ethers.utils.keccak256()
  const innerHash = keccak256(innerData);

  // 3. keccak256(inner_hash)
  // USE keccak256() DIRECTLY
  const leafNode = keccak256(innerHash);
  
  return leafNode;
}

// This is your entire reward list for the *one* cycle
const rewardList = [
  {
    recipient: "0x32822b10d94a6b117c2447d08d3e424360e2055c",
    cumulativeAmount: "100000000000000000000", // 100 tokens
    positionType: 1,
    positionId: "0x0000000000000000000000000000000000000000000000000000000000000001",
    cycle: 318,
    metadata: "0x"
  },
  {
    recipient: "0xc051134f56d56160e8c8ed9bb3c439c78ab27ccc", // (This is a target address we want a proof for)
    cumulativeAmount: "50000000000000000000",  // 50 tokens
    positionType: 2,
    positionId: "0x0000000000000000000000001a996cb54bb95462040408c06122d45d6cdb6096",
    cycle: 318,
    metadata: "0x"
  },
  {
    recipient: "0x1db3439a214f93644b15026932fc528651c50260",
    cumulativeAmount: "75000000000000000000",  // 75 tokens
    positionType: 1,
    positionId: "0x0000000000000000000000000000000000000000000000000000000000000003",
    cycle: 318,
    metadata: "0x"
  }
];

// Create all leaves
const leaves = rewardList.map(entry => createLeaf(entry));

// Build the tree.
// Pass the v6 keccak256 function directly
// INSTEAD OF ethers.utils.keccak256
const tree = new MerkleTree(leaves, keccak256, { sortPairs: true });

const root = tree.getHexRoot();

console.log("Merkle Root:");
console.log(root);

// This 'root' is what you pass to `proposeRoot(root, ...)`

// We want the proof for the second person in our list
const userToGetProofFor = rewardList[1];

// 1. Generate their specific leaf
const userLeaf = createLeaf(userToGetProofFor);

// 2. Get the proof (an array of bytes32 hashes)
const proof = tree.getHexProof(userLeaf);

console.log(`\nProof for ${userToGetProofFor.recipient}:`);
console.log(proof);

```

the output is used on the PoC. the necessary data is already pasted into the PoC. just apply the diff:

```diff
diff --git a/src/test/strategies/FluidARBUSDCStrategy.t.sol b/src/test/strategies/FluidARBUSDCStrategy.t.sol
index 4d5ddaf..1d8cd5a 100644
--- a/src/test/strategies/FluidARBUSDCStrategy.t.sol
+++ b/src/test/strategies/FluidARBUSDCStrategy.t.sol
@@ -10,6 +10,16 @@ contract MockFluidARBUSDCStrategy is FluidARBUSDCStrategy {
     {}
 }
 
+interface IRewardDistributor {
+    function proposeRoot(bytes32 root_,bytes32 contentHash_,uint40 cycle_,uint40 startBlock_,uint40 endBlock_) external;
+    function approveRoot(bytes32 root_,bytes32 contentHash_,uint40 cycle_,uint40 startBlock_,uint40 endBlock_) external;
+    function claim(address recipient_,uint256 cumulativeAmount_,uint8 positionType_,bytes32 positionId_,uint256 cycle_,bytes32[] memory merkleProof_,bytes memory metadata_) external;
+}
+
+interface IERC20 {
+    function balanceOf(address user) external returns(uint256);
+}
+
 contract FluidARBUSDCStrategyTest is BaseStrategyTest {
     address public constant FLUID_USDC_VAULT = 0x1A996cb54bb95462040408C06122D45D6Cdb6096;
     address public constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
@@ -60,4 +70,62 @@ contract FluidARBUSDCStrategyTest is BaseStrategyTest {
         IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
     }
+
+    function test_fluidStrategyCantClaimRewards() public {
+        // fork test arbitrum block to fork 395303779
+        // which is the block before the original proposer propose cycle 318
+        uint256 amountToAllocate = 1000e6;
+        vm.startPrank(vault);
+        deal(testConfig.vaultAsset, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        // console.log(strategy);
+        // strategy address is 0xc051134f56d56160e8c8ed9bb3c439c78ab27ccc
+        // positionId in fluid = 0x0000000000000000000000001a996cb54bb95462040408c06122d45d6cdb6096 (vault address)
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
+        require(initialRealAssets > 0, "Initial real assets is 0");
+        vm.stopPrank();
+
+        // setup rewards. using the provided merklegen.js for the root & proof
+        address proposer = 0x4f104710f8d9F6EFB28c4b2f057554928Daa3a83;
+        address approver = 0x85dC44E0c3AfdFedCa52678bD4C000917C6597B2;
+        address fluidMerkleDistributor = 0x94312a608246Cecfce6811Db84B3Ef4B2619054E;
+        address fluidToken = 0x61E030A56D33e8260FdD81f03B162A79Fe3449Cd;
+        bytes32 merkleRoot = 0x58a8e723c4d9674cac04be2efed7a5f730621fed2f0ce4d49d66a83cabacebb7;
+
+        uint256 amount = 50e18; // 50 fluid
+        uint8 positionType = 2;
+        bytes32 positionId = 0x0000000000000000000000001a996cb54bb95462040408c06122d45d6cdb6096;
+        bytes32 contentHash = 0x0000000000000000000000000000000000000000000000000000000000000000;
+        uint40 cycle = 318;
+        uint40 startBlock = 395303780;
+        uint40 endBlock = 395303785;
+        bytes32[] memory proof = new bytes32[](2);
+        proof[0] = 0x0f62e3b343933e94a21b26a06f7565b13ef1ebb9f778392e93570c797ef7a4b8;
+        proof[1] = 0xb23b02e9053de9c4e7b7cd2557dc1c34babe76213b889754ae0a93bd6bacfe85;
+        bytes memory metadata;
+
+        // finish the cycle
+        vm.roll(block.number + 7);
+
+        // we call this on block 395303786. start + finish cycle 318
+        vm.prank(proposer);
+        IRewardDistributor(fluidMerkleDistributor).proposeRoot(merkleRoot, contentHash, cycle, startBlock, endBlock);
+        vm.prank(approver);
+        IRewardDistributor(fluidMerkleDistributor).approveRoot(merkleRoot, contentHash, cycle, startBlock, endBlock);
+
+        // create revert id
+        uint256 id = vm.snapshot();
+
+        // we claim cycle 318, by prank the strategy itself
+        vm.prank(strategy);
+        IRewardDistributor(fluidMerkleDistributor).claim(strategy, amount, positionType, positionId, cycle, proof, metadata);
+        assertEq(amount, IERC20(fluidToken).balanceOf(strategy));
+
+        // now we try using the claimReward of strategy
+        vm.revertTo(id);
+        IMYTStrategy(strategy).claimRewards();
+        // but this would return 0
+        assertEq(0, IERC20(fluidToken).balanceOf(strategy));
+    }
 }

```

we run on specific block fork `forge test --fork-url https://arbitrum.gateway.tenderly.co --fork-block-number 395303779 --mt test_fluidStrategyCantClaimRewards`

the result would be the strategy is having 50e18 fluid token as reward but by using the `MYTStrategy::claimRewards` that is not overrided, it cant be claimed as shown that the balance is still 0. effectively this reward would be lost.
