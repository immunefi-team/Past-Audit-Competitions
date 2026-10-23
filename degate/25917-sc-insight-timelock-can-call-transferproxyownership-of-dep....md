# 25917 - \[SC - Insight] Timelock can call transferProxyOwnership of Dep...

Submitted on Nov 21st 2023 at 01:59:51 UTC by @said for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25917

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Theft of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.
* Permanent freezing of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

It is mentioned that the goal of `Timelock` is to solve the trustless issue when DeGate need to modify the code and complete the upgrade. However, timelock is not restricted to execute `transferProxyOwnership` inside `DepositContractProxy` and `ExchangeProxy`.

## Impact

This will allow `Timelock` to change proxy ownership, then the new owner can execute upgrade and modify code without trough the designed DeGate timelock process. Basically breaking trustless assumption and violating the new upgrade design.

<https://github.com/degatedev/protocols/blob/180138015197c886ec3c87efa8bf0031b653359f/packages/loopring\\_v3/contracts/thirdparty/timelock/Timelock.sol#L63-L72>

```solidity
    function queueTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public returns (bytes32) {
        require(msg.sender == admin, "Timelock::queueTransaction: Call must come from admin.");
        require(eta >= getBlockTimestamp().add(delay), "Timelock::queueTransaction: Estimated execution block must satisfy delay.");

        bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
        queuedTransactions[txHash] = true;

        emit QueueTransaction(txHash, target, value, signature, data, eta);
        return txHash;
    }
```

It can be observed that `queueTransaction` allow any operation including `transferProxyOwnership` to `DepositContractProxy` and `ExchangeProxy`

## Risk Breakdown

Difficulty to Exploit: Critical

## Recommendation

Restrict the queueTransaction function inside `Timelock` by checking if it is providing `transferProxyOwnership` call to `DepositContractProxy` and `ExchangeProxy`. If yes, revert the operations

## References

<https://github.com/degatedev/protocols/commit/180138015197c886ec3c87efa8bf0031b653359f?utm\\_source=immunefi#commitcomment-132582143>

<https://github.com/degatedev/protocols/commit/180138015197c886ec3c87efa8bf0031b653359f?utm\\_source=immunefi#diff-83099129c62e972adeec76325e36fa907afaac960342ce8de1c451d033ac41ef>

## Proof of concept

PoC Scenario :

1. MultiSig create `transferProxyOwnership` via `queueTransaction` operation trough timelock process and providing non-timelock address.
2. All MultiSig owners sign the transaction and wait for the normal timelock time.
3. All MultiSig create `executeTransaction`for the timelock operation and sign the transaction id.
4. The `transferProxyOwnership` is executed and proxy owner changed to non-timelock.
5. the new owner can change proxy implementation without any notice and delay and can break or steal funds from users.

Setup empty foundry, and run provided test in gist.

Note, to RUN the PoC, add the following to `foundry.toml` :

```
evm_version = "shanghai"
```
