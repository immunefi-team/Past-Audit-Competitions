# #36340 \[SC-Insight] TimeBasedCollateralPool::\_resetAccountTokenStateIfApplicable does not adjust tok

**Submitted on Oct 30th 2024 at 08:37:16 UTC by @niroh for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36340
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Temporary freezing of funds within the TimeBasedCollateralPool for at least 48 hours

## Description

## Brief/Intro

The TimeBasedCollateralPool \_resetAccountTokenStateIfApplicable is called in \_releaseEligibleAccountTokens to handle the case of a pool reset happening prior to the release that should affect the user. The function first handles the case of the account having a pending unstake that was already handled on the contract level before reset was called. In this case, the epoc pending units are first redeemed from the relevant epoc ExitBalance, and afterwards, if the account has additional pool units, they are redeemed from the reset exit balance:

(From TimeBasedCollateralPool.sol line 949) \`\`\`solidity // 2. Process and purge all account pending unstake state. There are two possibilities: // a. A pending unstake is vested or unvested but no vested contract unstake was processed prior to pool // reset. That contract state was purged in \_resetPool(...), the account pool units still exist in the // vault, and we can release them via the standard "everything has been unstaked" logic below. // b. A pending unstake, was vested and the contract unstake was already processed at the time of pool reset. // In this case, the exchange rate was captured in tokenEpochExitBalances, and it's unsafe to process // this unstake any different than the standard release process. uint256 unstakeUnitsToRelease; { uint256 epoch = accountStateStorage.firstPendingUnstakeEpoch; if (epoch > 0) { uint256 currentEpoch = getCurrentEpoch(); if (epoch < currentEpoch) { // NB: This is case b. from above -- do not process contract-level unstakes that have not already been processed. (unstakeUnitsToRelease, \_tokensToRelease) = getAccountExitUnitsAndTokens( \_tokenAddress, epoch, accountStateStorage.firstPendingUnstakeUnits, false ); epoch = accountStateStorage.secondPendingUnstakeEpoch; if (\_nonZeroAndLessThan(epoch, currentEpoch)) { (uint256 units, uint256 tokens) = getAccountExitUnitsAndTokens( \_tokenAddress, epoch, accountStateStorage.secondPendingUnstakeUnits, false ); unstakeUnitsToRelease += units; \_tokensToRelease += tokens; } }

```
        accountStateStorage.firstPendingUnstakeEpoch &#x3D; 0;
        accountStateStorage.secondPendingUnstakeEpoch &#x3D; 0;

        accountStateStorage.firstPendingUnstakeUnits &#x3D; 0;
        accountStateStorage.secondPendingUnstakeUnits &#x3D; 0;
    }
}

// 3. Process reset exit units.
if (_unitsToRelease &#x3D;&#x3D; unstakeUnitsToRelease) {
```

\`\`\`

## Vulnerability Details

The problem is that is this scenario, while the user receives their releasable tokens for their pending epoc units, the ExitBalance struct is not updated and the released tokens/units are not substracted from it. This is inconistent with the way release is handled in the regular case (without a following reset), where the ExitBalance leftTokens/leftUnits are reduced with each account release: (From TimeBasedCollateralPool.sol line 845, \_processAccountTokenUnstakes function) \`\`\`solidity /\*\* Update epoch-exit state \*\*/ tokenEpochExitBalances\[\_tokenAddress]\[epoch].unitsLeft -= \_unitsToRelease; tokenEpochExitBalances\[\_tokenAddress]\[epoch].tokensLeft -= \_tokensToRelease; \`\`\`

One of the sideeffects of this inconsistency in release calculations is that while the regular release process guarantees that all the exit balance tokens for that epoc will be distributed to users , in the reset case some residual exit balance remains unclaimable and permanently locked in the TBCP's vault account. The reason is that in the regular release process each user gets their share calculated from the reminder, up to the last user that gets all remaining tokens. In the reset case however, each user's share is calculated from the overall exit balance units/tokens. This causes rounding-down inacuraccies to accumulate and causes a situation that even after all units are claimed, not all exit balance tokens are transferred to the claimers (as vault available balance). Since the entire exit amount is released to the TBCP available balance, these residual tokens will remain in the TBCP vault available balance and will be locked there permanently.

## Impact Details

Some residual tokens remain locked in the CollateralVault as residual TBCP available balance, accumulating over time.

## References

[Release without a reset](https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L845) [Release after a reset](https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L947)

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
7. run \`forge test -vvv --match-test testReleaseAfterReset\`
8. Run once with the with\_reset variable set to true and once as false to see the different effect.

### test file code

\`\`\`solidity pragma solidity >=0.8.0;

import {Test} from "@forge-std/Test.sol"; import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol"; import {UpgradeableBeacon} from "@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol"; import {console2} from "@forge-std/console2.sol"; import "./../src/CollateralVault.sol"; import "./../src/TimeBasedCollateralPool.sol"; import "./../src/VisibleBeaconProxy.sol"; import "@openzeppelin/contracts/utils/Strings.sol";

contract TestReleaseAfterReset is Test {

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
    for (uint i &#x3D; 0;i&lt;300;i++) {
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

function testReleaseAfterReset() public {
    bool with_reset &#x3D; true;    


    //deposit some amount of USDC from all users
    uint256 amount &#x3D; 1245413485; // ($1245.4)
    for (uint i&#x3D;0;i&lt;users.length;i++) {
        poolerDepositAndStake(USDC,amount, basePriv+i,0);
    }

    //unstake from most of the stakers to simulate the release 
    for (uint i&#x3D;0;i&lt;users.length * 99 / 100;i++) {
        vm.prank(users[i]);
        tbcp.unstake(IERC20(USDC),amount );
    }

    //claim from pool (to make unit value more interesting)
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

    //warp time 4 weeks to vest the unstakes
    vm.warp(block.timestamp + 8 weeks);

    //call release just from the first user to invoke the contract level unstake
    tbcp.releaseEligibleTokens(users[0],tokenss);

    //reset pool
    if (with_reset) {
        vm.prank(resetter);
        tbcp.resetPool(tokenss);     
    }
   

    //release all user elligable tokens
    //(For the unstaking users, release will be calculated from the unstake exit balance
    //and for the rest of the users release will be calculated from the pool reset)
   for (uint i&#x3D;0;i&lt;users.length;i++) {
        vm.prank(users[i]);
        tbcp.releaseEligibleTokens(users[i],tokenss);
    }

    uint256 tbcpVaultBalance &#x3D; colVault.getAccountCollateralBalance(address(tbcp),USDC).available;
    console2.log(&quot;With reset: %s. TBCP vault available balance after all user release eligable tokens: %s&quot;,with_reset, tbcpVaultBalance);

    //Note that with the call to resetPool, some residual tokens remain as TBCP available balance even though a full release should not leave any.
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
