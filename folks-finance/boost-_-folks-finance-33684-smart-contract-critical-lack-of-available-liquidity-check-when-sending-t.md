# Boost \_ Folks Finance 33684 - \[Smart Contract - Critical] Lack of available liquidity check when sen

Submitted on Fri Jul 26 2024 10:31:34 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33684

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

When users perform `borrow` or `withdraw` operation, tokens are sent back via `sendTokenToUser` in `Hub` contract. The `Hub` contract then decides on how it should be sent (on which adapters to use).

However, there is no validation whether the actual available liquidity is sufficient on the destination Spoke endpoint in any steps of the execution flow.

```
File: /contracts/hub/Hub.sol
function sendTokenToUser(
    uint16 adapterId,
    uint256 gasLimit,
    bytes32 accountId,
    bytes32 recipient,
    SendToken memory sendToken
) internal {
    // generate message to send token
    IHubPool pool = loanManager.getPool(sendToken.poolId);
    Messages.MessageToSend memory messageToSend = pool.getSendTokenMessage(
        bridgeRouter,
        adapterId,
        gasLimit,
        accountId,
        sendToken.chainId,
        sendToken.amount,
        recipient
    );

    // send message (balance for user account already present in bridge router)
    _sendMessage(messageToSend, 0);
}

File: /contracts/hub/HubPool.sol
function getSendTokenMessage(
    IBridgeRouter bridgeRouter,
    uint16 adapterId,
    uint256 gasLimit,
    bytes32 accountId,
    uint16 chainId,
    uint256 amount,
    bytes32 recipient
) external override onlyRole(HUB_ROLE) nonReentrant returns (Messages.MessageToSend memory) {
    // check chain is compatible
    bytes32 spokeAddress = getChainSpoke(chainId);

    // prepare message
    Messages.MessageParams memory params = Messages.MessageParams({
        adapterId: adapterId,
        returnAdapterId: 0,
        receiverValue: 0,
        gasLimit: gasLimit,
        returnGasLimit: 0
    });
    bytes memory extraArgs = _sendToken(bridgeRouter, spokeAddress, params, amount);

    // construct message (will be sent from Hub)
    return
        Messages.MessageToSend({
            params: params,
            sender: Messages.convertEVMAddressToGenericAddress(msg.sender),
            destinationChainId: chainId,
            handler: spokeAddress,
            payload: Messages.encodeMessagePayload(
                Messages.MessagePayload({
                    action: Messages.Action.SendToken,
                    accountId: accountId,
                    userAddress: recipient,
                    data: abi.encodePacked(amount)
                })
            ),
            finalityLevel: 1, // finalised
            extraArgs: extraArgs
        });
}
```

To better illustrate, consider this situation:\
**ALICE** deposits 1\_000 USDC from Spoke Chain\
**BOB** borrows 1\_000 USDC from Spoke Chain\
As a result, the total balanceOf USDC on Spoke Chain is `0`.\
`depositData.totalAmount` is 1\_000 and `variableBorrowData.totalAmount` is 1\_000, stored on Hub Chain.

However, due to the lack of checking on available liquidity on Spoke Chain.\
**EVE** can proceed to call `borrow` function from Spoke Chain, let's say **EVE** call `borrow` with `amount=1_000`.

Although the message being executed on Spoke Chain will fail, the state is already changed and written on Hub Chain. So, `depositData.totalAmount` is 1\_000 and `variableBorrowData.totalAmount` is 1\_500, stored on Hub Chain.

Also, note that **EVE** can retry the fail message to get her token back.

As a consequence, this breaks the utilisation ratio invariance (0 <= U <= 1), as it is evidenced from the above situation.

The utilisation ratio after **EVE** transaction would be `1500/1000=1.5`.

And if the utilisation ratio breaks, it in turn breaks the interest rate.

This opens up an attack vector for first deposit attack and inflation attack.\
Imagine this attack scenario when the pool is just deployed:

1. **ALICE** deposits 100 USDC to get some borrowing power
2. **ALICE** forced transfers 1 AVAX to **SpokeGasToken** endpoint.
3. **ALICE** borrows 1 AVAX
4. **BOB** deposits 1 wei of AVAX (**BOB** gets 1 wei of **fAVAX**)

The final state from these transactions would look like this:

| **fAVAX**           |      |
| ------------------- | ---- |
| deposit.totalAmount | 1    |
| borrow\.totalAmount | 1e18 |

Utilisation ratio would be `1e18/1=1e18` which is astronomically larger than maxiumum `1`

This would in turn largely inflate deposit interest rate. Even within the next block on the Avalanche Chain, which has a block time of 2 seconds, the value of **BOB**'s **fAVAX** in 1 wei will become extremely expensive from deposit interest, enabling him to borrow anything.

And due to the lack of avaialble liquidity validation, **BOB** can also preemptively borrow other tokens on Spoke Chain and subsequently withdraw them by calling `retryMessage` upon a Spoke Chain new deposit.

*Note that in this case BOB and ALICE are the same person*

## Impact

* Attacker can leverage this vulnerability to drain all tokens in the pool.

## Recommended Mitigations

* Make use of `MathUtils#calcAvailableLiquidity` before sending token back.

## Proof of concept

## Proof-of-Concept

A test file in the secret gist demonstrates the attack scenario mentioned in `Description` section.\
The test is done on the old block (exactly after pools are set up) in order to accurately simulate first deposit attack.

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksBorrow-same.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/615d62aea2e0137e03cb44c18646113e>
4. Run `forge t --match-contract FolksBorrowSameTest -vv`
5. Observe that **BOB** ends up with 100\_000 USDC in his pocket

**Expected result**

```
Ran 1 test for test/FolksBorrow-same.t.sol:FolksBorrowSameTest
[PASS] test_firstDepositAndInflationAttack() (gas: 3757427)
Logs:
  --- Initial balance of Attacker
  ALICE: USDC=1e8 AVAX=1e18
  BOB: USDC=0e0 AVAX=1e18
  --- Stating of Block 1 ---
  @> BYSTANDER deposits 100_000 USDC to USDC pool
  @> ALICE deposits 100 USDC to get some borowing power
  @> ALICE forced tranfers 1 AVAX to SpokeGasToken
  @> ALICE borrows 1 AVAX to inflate the utilisation rate
  @> BOB deposits 1 wei of AVAX
  @> BOB calls deposit again to update deposit interest rate
  @> Final state of Block 1
  @> fAVAX-depositData.totalAmount=1
  @> fAVAX-depositData.interestIndex=1000000000000000000
  @> fAVAX-depositData.interestRate=5142857142857142853863000000000000000000000000000000000
  @> fAVAX-variableBorrow.totalAmount=5714285714285714282070000000000000000
  @> fAVAX-variableBorrow.interestRate=5714285714285714282070000000000000000
  @> Warping by 2 seconds
  --- Stating of Block 2 ---
  @> fAVAX state after updated interest index
  @> fAVAX depositData.totalAmount=1
  @> fAVAX depositData.interestIndex=326157860404435746693493150685931506849315068493
  --- BOB starts the attack ---
  @> BOB borrows 100_000 USDC from the pool
  @> BOB successfully borrowed 100_000 USDC with only 1 wei of AVAX
  --- Final balance of Attacker
  ALICE: USDC=0e0 AVAX=1e18
  BOB: USDC=1e11 AVAX=9.99999999999999999e17
```
