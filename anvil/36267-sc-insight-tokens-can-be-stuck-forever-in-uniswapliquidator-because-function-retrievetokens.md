# #36267 \[SC-Insight] tokens can be stuck forever in uniswapliquidator because function retrievetokens

## #36267 \[SC-Insight] Tokens can be stuck forever in UniswapLiquidator because function retrieveTokens always reverts for USDT and all tokens that have transfer function that do not return boolean

**Submitted on Oct 27th 2024 at 04:32:36 UTC by @perseverance for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36267
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://immunefi.com/>
* **Impacts:**
  * Permanent freezing of funds

### Description

## Description

### Brief/Intro

This bug is reported for contract UniswapLiquidator that is **Primacy of Impact** scope as this bug has severity of **Critical** and impact that is in scope is: **Permanent freezing of funds**

The contract UniswapLiquidator is used for liquidation process and the address is listed as deployement address:

\`\`\` UniswapLiquidator 0xe358594373B4C7D268204f3D1E5226ce4dB2A712 \`\`\` Reference: <https://github.com/AcronymFoundation/anvil-contracts?utm\\_source=immunefi>

This contract usually can receive tokens that are supported by the protocol for the liquidation process. As informed by the protocol team, some tokens supported by the protocol are: WETH, USDC, USDT . In the case, there are some tokens received because someone sent errorneously to this contract, then the owner can use function retrieveTokens to tranfer tokens. This is seen in the comments of the retrieveTokens function. This situation is likely to happen because this contract usually receive tokens and that is the reason the Anvil team have designed this retrieveTokens function.

\`\`\`solidity /\*\* \* @notice Enable the contract owner to send erroneously received tokens to a specified recipient \* @param \_token The token to transfer \* @param \_recipient The recipient of the transferred tokens \* @param \_amount The amount of tokens to transfer \*/ function retrieveTokens(IERC20 \_token, address \_recipient, uint256 \_amount) external onlyOwner { \_token.transfer(\_recipient, \_amount); }

\`\`\`

### The vulnerability

#### Vulnerability Details

The vulnerability here is that the action to transfer tokens in retrieveTokens is using the function **transfer**

But the transfer interface of IERC20 do require the token to return a boolean value as in the IERC20 interface code

\`\`\`solidity import "@openzeppelin/contracts/token/ERC20/IERC20.sol"; \`\`\`

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol#L41>

\`\`\`solidity /\*\* \* @dev Moves a \`value\` amount of tokens from the caller's account to \`to\`. \* \* Returns a boolean value indicating whether the operation succeeded. \* \* Emits a {Transfer} event. \*/ function transfer(address to, uint256 value) external returns (bool);

\`\`\`

It is important to note that the transfer functions of some tokens (e.g., USDT, BNB) do not return any values, so these tokens are incompatible with the current version of the contract. The transaction retrieveTokens will always revert for USDT and any tokens that have transfer function that do not return a boolean value. Please also note that USDT is supported by the Anvil Protocol now.

**So if USDT is stuck in the contract, the USDT will be frozen forever. There is no way to retrieve the tokens out.**

USDT contract: <https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code>

<https://vscode.blockscan.com/ethereum/0xdac17f958d2ee523a2206206994597c13d831ec7>

\`\`\`solidity function transfer(address \_to, uint \_value) public onlyPayloadSize(2 \* 32) { uint fee = (\_value.mul(basisPointsRate)).div(10000); if (fee > maximumFee) { fee = maximumFee; } uint sendAmount = \_value.sub(fee); balances\[msg.sender] = balances\[msg.sender].sub(\_value); balances\[\_to] = balances\[\_to].add(sendAmount); if (fee > 0) { balances\[owner] = balances\[owner].add(fee); Transfer(msg.sender, owner, fee); } Transfer(msg.sender, \_to, sendAmount); } \`\`\`

## Impacts

## About the severity assessment

Bug Severity: Critical

Impact category: Permanent freezing of funds

As designed and assessed by the protocol team that the contract UniswapLiquidator can receive tokens as USDT. If there are tokens USDT in the contract, owner can use function retrieveTokens to rescue the tokens. But with this bug, the USDT will be frozen forever in the contract. There is no way to rescue the tokens.

The scenarity is likely to happen.

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/bd288749bb72d7b438b6e206c0a961e5#file-testuniswapliquidator-sol>

### Proof of Concept

### Proof of concept

POC to prove the bug

Step 1: Simulate the situation that there is 1000 USDT stuck in the contract UniswapLiquidator

Step 2: the Owner call function retrieveTokens to rescue the tokens. But the transaction reverts.

POC Code: \`\`\`solidity function testRetrieveTokens() public {

```
    console.log(&quot;Simulate the situation that someone erroneously send USDT to the contract UniswapLiquidator&quot;);    
    console.log(&quot;Balance of USDT of UniswapLiquidator before now: &quot;,IERC20(USDT).balanceOf(UniswapLiquidator_contract));
    USDT_Token(USDT).transfer(UniswapLiquidator_contract, 1000*10**6);
    console.log(&quot;Balance of USDT of UniswapLiquidator after someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDT).balanceOf(UniswapLiquidator_contract)); 
    
    console.log(&quot;Owner will try to retrieve the USDT tokens from the contract UniswapLiquidator by calling retrieveTokens&quot;);
    vm.startPrank(Owner);
    UniswapLiquidator(UniswapLiquidator_contract).retrieveTokens(IERC20(USDT), Owner, 1000*10**6); 
    vm.stopPrank();     
    console.log(&quot;Balance of USDT of UniswapLiquidator after rescue someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDT).balanceOf(UniswapLiquidator_contract)); 
    console.log(&quot;Balance of USDT of Owner after srescue: &quot;,IERC20(USDT).balanceOf(Owner)); 
    
} 
```

\`\`\`

Test log:

\`\`\`Log \[FAIL. Reason: EvmError: Revert] testRetrieveTokens() (gas: 97962) Logs: Simulate the situation that someone erroneously send USDT to the contract UniswapLiquidator Balance of USDT of UniswapLiquidator before now: 0 Balance of USDT of UniswapLiquidator after someone erroneously sent USDT to the contract UniswapLiquidator: 1000000000 Owner will try to retrieve the USDT tokens from the contract UniswapLiquidator by calling retrieveTokens

Traces: \[353376] testUniswapLiquidator::setUp() ├─ \[0] VM::createSelectFork("<https://rpc.ankr.com/eth>", 21053617 \[2.105e7]) │ └─ ← \[Return] 0

├─ \[31722] 0xe358594373B4C7D268204f3D1E5226ce4dB2A712::retrieveTokens(0xdAC17F958D2ee523a2206206994597C13D831ec7, 0xAf000d2594DfEA1d04693243311d2D71e8879A06, 1000000000 \[1e9]) │ ├─ \[28801] 0xdAC17F958D2ee523a2206206994597C13D831ec7::transfer(0xAf000d2594DfEA1d04693243311d2D71e8879A06, 1000000000 \[1e9]) │ │ ├─ emit Transfer(from: 0xe358594373B4C7D268204f3D1E5226ce4dB2A712, to: 0xAf000d2594DfEA1d04693243311d2D71e8879A06, value: 1000000000 \[1e9]) │ │ └─ ← \[Stop] │ └─ ← \[Revert] EvmError: Revert └─ ← \[Revert] EvmError: Revert

\`\`\`

I also created the similar situation for USDC. The owner can rescue the tokens successfully.

Test code: \`\`\`solidity // forge test --match-test testRetrieveTokens -vvvvv | format > testRetrieveTokens\_241027\_0800.log function testRetrieveTokens\_success() public {

```
    console.log(&quot;Simulate the situation that someone erroneously send USDC to the contract UniswapLiquidator&quot;);    
    console.log(&quot;Balance of USDT of UniswapLiquidator before now: &quot;,IERC20(USDC).balanceOf(UniswapLiquidator_contract));
    IERC20(USDC).transfer(UniswapLiquidator_contract, 1000*10**6);
    console.log(&quot;Balance of USDT of UniswapLiquidator after someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDC).balanceOf(UniswapLiquidator_contract)); 
    
    console.log(&quot;Owner will try to retrieve the USDC tokens from the contract UniswapLiquidator by calling retrieveTokens&quot;);
    vm.startPrank(Owner);
    UniswapLiquidator(UniswapLiquidator_contract).retrieveTokens(IERC20(USDC),  Owner, 1000*10**6); 
    vm.stopPrank();  
    console.log(&quot;Balance of USDT of UniswapLiquidator after rescue someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDC).balanceOf(UniswapLiquidator_contract)); 
    console.log(&quot;Balance of USDT of Owner after srescue: &quot;,IERC20(USDC).balanceOf(Owner)); 
    
} 
```

\`\`\`

Test log: \`\`\` \[PASS] testRetrieveTokens\_success() (gas: 77524) Logs: Simulate the situation that someone erroneously send USDC to the contract UniswapLiquidator Balance of USDC of UniswapLiquidator before now: 0 Balance of USDC of UniswapLiquidator after someone erroneously sent USDT to the contract UniswapLiquidator: 1000000000 Owner will try to retrieve the USDC tokens from the contract UniswapLiquidator by calling retrieveTokens Balance of USDC of UniswapLiquidator after rescue someone erroneously sent USDT to the contract UniswapLiquidator: 0 Balance of USDC of Owner after srescue: 1000000000 \`\`\`

Test code full:

<https://gist.github.com/Perseverancesuccess2021/bd288749bb72d7b438b6e206c0a961e5#file-testuniswapliquidator-sol>

\`\`\`solidity // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.0;

import "forge-std/Test.sol"; import "forge-std/console.sol"; import "../src/liquidation/UniswapLiquidator.sol";

interface USDT\_Token { function transfer(address to, uint256 value) external; }

contract testUniswapLiquidator is Test {

```
address USDT &#x3D; 0xdAC17F958D2ee523a2206206994597C13D831ec7; 
address USDC &#x3D; 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48; 
address UniswapLiquidator_contract &#x3D; 0xe358594373B4C7D268204f3D1E5226ce4dB2A712; 
address Owner &#x3D; 0xAf000d2594DfEA1d04693243311d2D71e8879A06; 

function setUp() public {
    vm.createSelectFork(&quot;https://rpc.ankr.com/eth&quot;, 21053617); // Oct-27-2024 01:16:23 AM +UTC 
    deal(USDT,address(this), 1000*10**6); 
    deal(USDC,address(this), 1000*10**6); 
   
   
}

// forge test --match-test testRetrieveTokens -vvvvv | format &gt; testRetrieveTokens_241027_0800.log
function testRetrieveTokens() public {
   
    console.log(&quot;Simulate the situation that someone erroneously send USDT to the contract UniswapLiquidator&quot;);    
    console.log(&quot;Balance of USDT of UniswapLiquidator before now: &quot;,IERC20(USDT).balanceOf(UniswapLiquidator_contract));
    USDT_Token(USDT).transfer(UniswapLiquidator_contract, 1000*10**6);
    console.log(&quot;Balance of USDT of UniswapLiquidator after someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDT).balanceOf(UniswapLiquidator_contract)); 
    
    console.log(&quot;Owner will try to retrieve the USDT tokens from the contract UniswapLiquidator by calling retrieveTokens&quot;);
    vm.startPrank(Owner);
    UniswapLiquidator(UniswapLiquidator_contract).retrieveTokens(IERC20(USDT), Owner, 1000*10**6); 
    vm.stopPrank();     
    console.log(&quot;Balance of USDT of UniswapLiquidator after rescue someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDT).balanceOf(UniswapLiquidator_contract)); 
    console.log(&quot;Balance of USDT of Owner after srescue: &quot;,IERC20(USDT).balanceOf(Owner)); 
    
} 

// forge test --match-test testRetrieveTokens -vvvvv | format &gt; testRetrieveTokens_241027_0800.log
function testRetrieveTokens_success() public {
   
    console.log(&quot;Simulate the situation that someone erroneously send USDC to the contract UniswapLiquidator&quot;);    
    console.log(&quot;Balance of USDC of UniswapLiquidator before now: &quot;,IERC20(USDC).balanceOf(UniswapLiquidator_contract));
    IERC20(USDC).transfer(UniswapLiquidator_contract, 1000*10**6);
    console.log(&quot;Balance of USDC of UniswapLiquidator after someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDC).balanceOf(UniswapLiquidator_contract)); 
    
    console.log(&quot;Owner will try to retrieve the USDC tokens from the contract UniswapLiquidator by calling retrieveTokens&quot;);
    vm.startPrank(Owner);
    UniswapLiquidator(UniswapLiquidator_contract).retrieveTokens(IERC20(USDC),  Owner, 1000*10**6); 
    vm.stopPrank();  
    console.log(&quot;Balance of USDC of UniswapLiquidator after rescue someone erroneously sent USDT to the contract UniswapLiquidator: &quot;,IERC20(USDC).balanceOf(UniswapLiquidator_contract)); 
    console.log(&quot;Balance of USDC of Owner after srescue: &quot;,IERC20(USDC).balanceOf(Owner)); 
    
} 
```

}

\`\`\`

#### To run test:

Just download the zip file:

<https://drive.google.com/file/d/1ga9D-KIgOkuBhIBpeUBatoWI\\_0mUgGIS/view?usp=sharing>

The Test use Foundry.

Unzip and run the test case:

\`\`\`bash forge test --match-test testRetrieveTokens -vvvvv > testRetrieveTokens\_241027\_0800.log

\`\`\`
