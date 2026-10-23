# #42127 \[SC-Insight] Redundant Fee Calculation in addYeetback() function

**Submitted on Mar 21st 2025 at 02:59:35 UTC by @Oxblackadam for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42127
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeetback.sol>
* **Impacts:**

## Description

## Brief/Intro

The restart function in the Yeet contract and the addYeetback function in the Yeetback contract both calculate the entropy fee, resulting in redundant operations and unnecessary gas consumption.

## Vulnerability Details

In the restart function, yeetback.getEntropyFee() is called to get the fee and also verify that sufficient ETH is provided to cover the entropy fee. Later, when calling yeetback.addYeetback, the fee is passed along with the potToYeetback amount. Inside the addYeetback function, getEntropyFee() is called again before using the fee to request entropy.\
This double calculation is unnecessary and consumes additional gas. The fee calculation should be performed once and then reused.

## code Yeet.sol

```solidity
function restart(bytes32 userRandomNumber) external payable whenNotPaused {
    if (userRandomNumber == bytes32(0)) {
        revert InvalidRandomNumber();
    }

    if (!isRoundFinished()) {
        revert RoundStillLive(roundNumber);
    }

    if (!hasCooldownEnded()) {
        revert CooldownNotEnded(block.timestamp, endOfYeetTime + COOLDOWN_TIME);
    }

    emit RoundWinner(lastYeeted, block.timestamp, potToWinner, roundNumber, nrOfYeets);

    uint256 fee = yeetback.getEntropyFee();
    if (msg.value < fee) {
        revert NotEnoughValueToPayEntropyFee(msg.value, fee);
    }
    uint256 remaining = msg.value - fee;

    if (potToYeetback > 0) {
        yeetback.addYeetback{value: fee + potToYeetback}(userRandomNumber, roundNumber, potToYeetback);
    }

    winnings[lastYeeted] += potToWinner;

    _roundWinners[roundNumber] = Winner(lastYeeted, block.timestamp, potToWinner, roundNumber);

    copySettings();
    roundNumber += 1;
    potToYeetback = 0;
    potToWinner = 0;
    nrOfYeets = 0;
    lastYeeted = address(0);
    lastYeetedAt = 0;
    yeetTimeInSeconds = YEET_TIME_SECONDS;
    endOfYeetTime = block.timestamp + yeetTimeInSeconds + BOOSTRAP_PHASE_DURATION;
    roundStartTime = block.timestamp;

    if (remaining > 0) {
        (bool success,) = payable(msg.sender).call{value: remaining}("");
        require(success, "Transfer failed, cant return remaining value to sender");
    }

    emit RoundStarted(
        roundNumber,
        roundStartTime,
        YEET_TIME_SECONDS,
        POT_DIVISION,
        TAX_PER_YEET,
        TAX_TO_STAKERS,
        TAX_TO_PUBLIC_GOODS,
        TAX_TO_TREASURY,
        YEETBACK_PERCENTAGE,
        COOLDOWN_TIME
    );
}
```

Since `amount` (potToYeetback) is passed along with the addYeetback function argument.\
here

```solidity
            yeetback.addYeetback{value: fee + potToYeetback}(userRandomNumber, roundNumber, potToYeetback);
```

In the `addYeetback` function, the fee can be computed as msg.value - amount (no redundant call) instead of `getEntropyFee`, saving the caller some gas

## code Yeetback.sol

```solidity
    function addYeetback(bytes32 userRandomNumber, uint256 round, uint256 amount) public payable onlyOwner {
        require(userRandomNumber != bytes32(0), "Invalid number");
        require(round != 0, "Invalid round");
        require(amount != 0, "Invalid amount");
        potForRound[round] = amount;

        uint256 fee = getEntropyFee();
        uint64 sequenceNumber = entropy.requestWithCallback{value: fee}(entropyProvider, userRandomNumber);
        sequenceToRound[sequenceNumber] = round;
        emit RandomNumberRequested(sequenceNumber);
        emit YeetbackAdded(round, amount);
    }
```

## Impact Details

This inefficiency increases the gas cost for users who call the restart function. While not a security vulnerability, it represents a suboptimal implementation that costs users additional gas.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

In `yeet.test.sol`, under the `Yeet_Claim` contract, Copy and paste the following test and run it:

* `forge test --mt test_gas_Usage -vvvv`

```solidity
function test_gas_Usage() public {
        yeet.yeet{value: 1 ether}();

        skip(2 hours);
        bytes32 randomNumber = 0x3b67d060cb9b8abcf5d29e15600b152af66a881e8867446e798f5752845be90d;

        uint128 fee = yeet.yeetback().getEntropyFee();

        uint256 startGas = gasleft();
        yeet.restart{value: fee}(randomNumber);
        uint256 endGas = gasleft();

        console.log("Gas used for restart:", startGas - endGas);
}
```

**Results:**

```
├─ [0] console::log("Gas used for restart:", 239126 [2.391e5]) [staticcall]
```

The original `restart` function cost **239,126 gas**.

Now, edit the `addYeetback` function in `Yeetback.sol`, replacing:

```solidity
uint256 fee = getEntropyFee();  // redundant call
```

with:

```solidity
uint256 fee = msg.value - amount;
```

and ran the test again.

**New results:**

```
├─ [0] console::log("Gas used for restart:", 238149 [2.381e5]) [staticcall]
```

The modified `restart` function now costs **238,149 gas**, saving **977 gas**.
