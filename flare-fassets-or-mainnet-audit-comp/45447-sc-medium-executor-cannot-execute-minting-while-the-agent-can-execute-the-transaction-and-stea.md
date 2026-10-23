# #45447 \[SC-Medium] Executor cannot execute minting while the agent can execute the transaction and steal executor fee

**Submitted on May 14th 2025 at 21:31:23 UTC by @holydevoti0n for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45447
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/docs/ImmunefiScope.md>
* **Impacts:**
  * Permanent freezing of funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

An executor who is a smart contract is likely to receive DoS when calling `MintingFacet.executeMinting` due to a gas limit set for native transfer. This will not only cause permanent DoS of the executor but also the loss of funds of his fees as the agent can call `executeMinting` and the contract will transfer those fees to the agent's collateral pool.

## Vulnerability Details

The vulnerability exists in the `executeMinting` function flow, specifically in how native token transfers are handled:

When a user reserves makes a collateral reservation, they provide a fee for the executor: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol#L70-L71>

```solidity
function reserveCollateral(...){ 
       ...
       cr.executor = _executor;
       cr.executorFeeNatGWei = ((msg.value - reservationFee) / 
       Conversion.GWEI).toUint64();
       ...
}
```

When the executor calls `executeMinting`, they should receive their fee: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/Minting.sol#L61-L65>

```solidity
uint256 unclaimedExecutorFee = crt.executorFeeNatGWei * Conversion.GWEI;
if (msg.sender == crt.executor) {
    Transfers.transferNAT(crt.executor, unclaimedExecutorFee);
    unclaimedExecutorFee = 0;
}
```

However, the `transferNAT` function imposes a strict gas limit of 100,000 for the ETH transfer:

```solidity
(bool success, ) = _recipient.call{value: _amount, gas: TRANSFER_GAS_ALLOWANCE}("");
```

If the executor is a smart contract that performs any operations in its `receive` function that exceed this gas limit, the transfer will fail with "transfer failed" error.

For instance, a simple ERC20 transfer could cost around 50k \~ 100k in gas. <https://ethereum.stackexchange.com/questions/41763/how-much-gas-does-an-erc20-transfer-cost>

When the transfer fails, the executor not only loses the gas spent on the transaction but also their rightful fee, as the unclaimed fee gets redirected to the agent's collateral pool: <https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/Minting.sol#L67-L68>

```solidity
CollateralReservations.distributeCollateralReservationFee(agent,
        crt.reservationFeeNatWei + unclaimedExecutorFee);
```

Additionally, the agent can also call `executeMinting` as they satisfy the permission check:

```solidity
require(msg.sender == crt.minter || msg.sender == crt.executor || Agents.isOwner(agent, msg.sender),
```

This creates an incentive for agents to front-run executor transactions. Since agents can also call `executeMinting` and would directly benefit from the executor's fees being added to the collateral pool, they are financially motivated to ensure executors fail to claim their fees.

## Impact Details

* Executors that are smart contracts can experience permanent DoS when attempting to execute minting on user's behalf, resulting in financial losses from both transaction gas costs and unclaimed executor fees.
* This causes the executor fees to be forcibly redirected to the agent's collateral pool, creating an unfair economic advantage for agents.
* The system incentivizes malicious behavior where agents can front-run executor transactions to capture executor fees.

## Recommendations

As the executor is the one calling the transaction, he should not have any limitation of gas. Additionally, rethink the logic of donating the executor fee to the protocol(as the agent can call this function, creating a conflict of interest).

## Proof of Concept

The PoC below shows how a simple smart contract can surpass the 100k gas limit and cause the transaction to revert.

Setup:

* Install foundry by:

1. `yarn add @nomicfoundation/hardhat-foundry`
2. Add `import "@nomicfoundation/hardhat-foundry";`on hardhat config file
3. Run `npx hardhat init-foundry` and `forge install foundry-rs/forge-std`
4. In `foundry.toml` update the `test` value to: `'test/foundry'`

* Create the test:

1. Create a folder called `foundry` under the `test` folder
2. Create a file called `Transfer.t.sol` in the `test/foundry` folder and paste the code below. Then run: `forge test --match-test testTransferNAT`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "forge-std/Test.sol";
import "../../contracts/utils/lib/Transfers.sol";
import "../../contracts/assetManager/mock/ERC20Mock.sol";
import "../../contracts/utils/mock/TransfersMock.sol";

// Make a contract that have a receive fallback
contract Executor {
    ERC20Mock public token;
    address public owner;
    address public transfersMock;

    constructor(address _owner, address _token, address _transfersMock) {
        owner = _owner;
        transfersMock = _transfersMock;
        token = ERC20Mock(_token);
        token.mintAmount(address(this), 1000e18);
    }

    receive() external payable {
        token.transfer(address(transfersMock), 10e18);
        token.transfer(address(owner), 10e18);
        token.withdraw(10e18);
    }
}

contract TransferTest is Test {
    TransfersMock transfersMock;
    ERC20Mock public token;
    function setUp() public {
        transfersMock = new TransfersMock();
        token = new ERC20Mock("Token", "TKN");

        // give NAT to TransfersMock and Token
        deal(address(transfersMock), 100e18);
        deal(address(token), 100e18);
    }
    function testTransferNAT() public {
        Executor executor = new Executor(address(this), address(token), address(transfersMock));

        // expect OutOfGas error
        vm.expectRevert("transfer failed");
        transfersMock.transferNAT(payable(address(executor)), 1e18);
    }
}
```

Output:

```javascript
Ran 1 test for test/foundry/Transfer.t.sol:TransferTest
[PASS] testTransferNAT() (gas: 533646)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.25ms (1.37ms CPU time)
```
