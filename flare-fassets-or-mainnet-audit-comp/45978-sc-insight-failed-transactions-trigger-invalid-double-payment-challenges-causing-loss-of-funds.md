# #45978 \[SC-Insight] Failed Transactions Trigger Invalid Double Payment Challenges Causing Loss of Funds for Legitimate Agents

**Submitted on May 23rd 2025 at 05:54:56 UTC by @Bluedragon for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45978
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Challenges.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary:

The `doublePaymentChallenge()` function in the FAssets system only validates payment reference matching between transactions but fails to verify that both transactions actually spent funds (amount > 0). This allows challengers to unfairly liquidate legitimate agents who retry failed redemption payments, as the system treats a failed transaction (spent amount = 0) and its successful retry (spent amount > 0) as duplicate payments.

## Vulnerability Details:

The vulnerability exists in the double payment challenge mechanism where the system validates duplicate payments by checking if two transactions share the same payment reference. However, it doesn't verify whether both transactions actually transferred funds (spend amount > 0).

The flow is challenger bot detects transactions with matching payment references and triggers challenges. The actual validation occurs in the `doublePaymentChallenge()` function called on the AssetManager contract.

The `doublePaymentChallenge()` function checks for duplicate payment references but does not check the spent amount, causing legitimate agents to be liquidated if they retry a failed payment. This is particularly problematic in scenarios where network congestion or other issues cause the first payment attempt to fail, but the agent retries with the same payment reference.

**Scenario step by step:**

1. **Agent receives redemption request**: Agent must pay redeemer on underlying chain (e.g., XRPL)
2. **First payment attempt fails**: Due to network congestion, insufficient gas, or other external factors, the transaction fails with spent amount = 0, but is recorded on-chain with the redemption payment reference
3. **Agent retries payment**: Agent makes a second payment with the same payment reference, which succeeds with spent amount > 0
4. **Challenger detects duplicate references**: The challenger bot detects both transactions have the same payment reference and triggers a double payment challenge
5. **System validates challenge**: The `doublePaymentChallenge()` function validates the challenge based solely on matching payment references, ignoring that the first transaction spent 0 funds and transaction status.
6. **Agent liquidated**: The legitimate agent enters full liquidation status and loses collateral.

## Impact:

* **Critical Fund Loss**: Legitimate agents can lose their entire collateral through full liquidation
* **System Instability**: Agents may avoid the system due to unfair liquidation risk
* **Economic Attack Vector**: Malicious actors could exploit network congestion to trigger failed transactions and subsequent challenges

## Recommended Mitigation:

Modify the `doublePaymentChallenge()` function to include spent amount validation:

1. **Add spent amount check**: Verify that both transactions have spent amount > 0 before considering them as valid duplicate payments
2. **Update challenge validation logic**: The function should reject challenges where either transaction has a failed transaction status or spent amount = 0

```solidity
    function doublePaymentChallenge(
        IBalanceDecreasingTransaction.Proof calldata _payment1,
        IBalanceDecreasingTransaction.Proof calldata _payment2,
        address _agentVault
    )
        internal
    {
        ....
        TransactionAttestation.verifyBalanceDecreasingTransaction(_payment1);
        TransactionAttestation.verifyBalanceDecreasingTransaction(_payment2);

        // check the payments are unique and originate from agent's address

        // @note custom mitigation
        require(_payment1.data.responseBody.spentAmount > 0 &&
            _payment2.data.responseBody.spentAmount > 0, "chlg dbl: zero spent amount");
        ....
    }
```

## Proof of Concept

## Proof of Concept:

1. Add the following custom logic to make the failed transaction to be recorded.

Replace the `createMultiTransaction` function in `MockChain.ts` with the following code:

```ts
createMultiTransaction(spent_: { [address: string]: BNish }, received_: { [address: string]: BNish }, reference: string | null, options?: MockTransactionOptions): MockChainTransaction {
        const inputs: TxInputOutput[] = Object.entries(spent_).map(([address, amount]) => [address, toBN(amount)]);
        const outputs: TxInputOutput[] = Object.entries(received_).map(([address, amount]) => [address, toBN(amount)]);
        const totalSpent = inputs.reduce((a, [_, x]) => a.add(x), BN_ZERO);
        const totalReceived = outputs.reduce((a, [_, x]) => a.add(x), BN_ZERO);
        const status = options?.status ?? TX_SUCCESS;
        if (status === TX_SUCCESS) {
            assert.isTrue(totalSpent.gte(totalReceived), "mockTransaction: received more than spent");
            assert.isTrue(totalSpent.gte(totalReceived.add(this.chain.requiredFee)), "mockTransaction: not enough fee");
        }
        const hash = this.chain.createTransactionHash(inputs, outputs, reference);
        // hash is set set when transaction is added to a block
        return { hash, inputs, outputs, reference, status };
    }
```

2. Add the following code to the `ChallengerTests.ts` in `test/unit/bots` directory
3. Run the test using `yarn testHH`

```ts
it.only("legitimate agents gets challenged on double payment", async () => {
  const challenger = new Challenger(runner, trackedState, challengerAddress1);
  await performMinting(minter, agent, 50);
  const [reqs] = await redeemer.requestRedemption(10);
  const paymentAmount = reqs[0].valueUBA.sub(reqs[0].feeUBA);
  const maxFee = chain.requiredFee.sub(toBN("1"));
  // 1st tx fails due to less gas or network congestion (real world scenario)
  const txHash = await agent.wallet.addTransaction(
    agent.underlyingAddress,
    reqs[0].paymentAddress,
    paymentAmount,
    reqs[0].paymentReference,
    { maxFee: maxFee }
  );
  const tx = chain.getTransaction(txHash);
  // Observe the status which is 1 (meeans tx failed)
  console.log("1st TX:", tx);
  // repeat the same payment (which succeeds now)
  const txHash2 = await agent.performRedemptionPayment(reqs[0]);
  const tx2 = chain.getTransaction(txHash);
  // Observe the status which is 0 (meeans tx succeed)
  console.log("2nd TX:", tx);
  // Bot queries the tx and challenges the legitimate agent
  await waitThreadsToFinish();
  // Observe the agent is fully liquidated
  const status = (await getAgentStatus(agent)).toString();
  console.log("Status of agent after challenge executed:", status);
  assert.equal(await getAgentStatus(agent), AgentStatus.FULL_LIQUIDATION);
});
```

**Logs**

```ts
  Contract: ChallengerTests.ts; test/unit/bots/ChallengerTests.ts; Challenger bot unit tests
1st TX: Promise {
  {
    hash: '0x77723f067fd0cb178284951b02ed3c251869f5e21fb48152d6cfce46f46e461b',
    inputs: [ [Array] ],
    outputs: [ [Array] ],
    reference: '0x4642505266410002000000000000000000000000000000000000000000000062',
    status: 1
  }
}
2nd TX: Promise {
  {
    hash: '0xf74795635b17e9bb07ccb392ae9f80a3bc4bd5c2436330e308740a7062476f4c',
    inputs: [ [Array] ],
    outputs: [ [Array] ],
    reference: '0x4642505266410002000000000000000000000000000000000000000000000062',
    status: 0
  }
}
Status of agent after challenge executed: 3
    ✔ legitimate agents gets challenged on double payment (493ms)
```

**Logs after implementing the fix**

```ts
    Contract: ChallengerTests.ts; test/unit/bots/ChallengerTests.ts; Challenger bot unit tests
1st TX: Promise {
  {
    hash: '0x77723f067fd0cb178284951b02ed3c251869f5e21fb48152d6cfce46f46e461b',
    inputs: [ [Array] ],
    outputs: [ [Array] ],
    reference: '0x4642505266410002000000000000000000000000000000000000000000000062',
    status: 1
  }
}
2nd TX: Promise {
  {
    hash: '0xf74795635b17e9bb07ccb392ae9f80a3bc4bd5c2436330e308740a7062476f4c',
    inputs: [ [Array] ],
    outputs: [ [Array] ],
    reference: '0x4642505266410002000000000000000000000000000000000000000000000062',
    status: 0
  }
}

Error: VM Exception while processing transaction: reverted with reason string 'chlg dbl: zero spent amount'
    at ChallengesFacet._checkOnlyWhitelistedSender (contracts/assetManager/facets/AssetManagerBase.sol:50)
    at ChallengesFacet.doublePaymentChallenge (contracts/assetManager/facets/ChallengesFacet.sol:49)
    at AssetManager.<fallback> (contracts/diamond/implementation/Diamond.sol:38)
```
