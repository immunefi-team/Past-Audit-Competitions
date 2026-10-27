# #45904 \[SC-High] Malicious agent can forge a non-payment proof despite user's valid payment and fraudulently trigger \`mintingPaymentDefault\`

**Submitted on May 22nd 2025 at 08:49:46 UTC by @nnez for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45904
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Details

### MintingDefault

After making a collateral reservation, minter (user) has to perform a payment on underlying chain to agent's designated address with a correct amount and payment reference in the memo (for XRP chain).

In addition, minter (user) must also perform a payment within the time limit, which is set at 15 minutes or 225 blocks for fXRP. After the time passes and minter/user fails to perform a correct payment in time, agent can request a proof of non-payment from FDCHub and call `CollateralReservationsFacet.mintingPaymentDefault` along with the proof to unstick their the reservation amount. It's also noteworthy that in the default case, the agent still takes the collateral reservation fee.

### The proof of non-existence payment

The request for obtain a proof of non-existence payment looks something like: <https://dev.flare.network/fdc/attestation-types/referenced-payment-nonexistence#request>

Basically, prover (user) specifies the range of search to look for the payment with exact amount, payment reference to designated destination address.\
The FDC client will query all transactions within specified range, then iterate for that payment, if the payment is not found, the request is confirmed (proven) that there is no transaction within range that satisfies the criteria. (See: <https://dev.flare.network/fdc/attestation-types/referenced-payment-nonexistence#verification-process>)

It is also optional to specify `checkSourceAddresses` and `sourceAddressesRoot` in addition to the criteria to search for a payment that comes from specified source address.

The latter option is available for collateral reservation that requires handshake (approval from agent). In other words, agent will only accept funds from certain sources in this mode. Therefore, the proof must include the verification of the source address, hence, the `checkSourceAddresses`.

If the collateral reservation is of type the requires handshaking, the source address is stored in `crt.sourceAddressesRoot`. Otherwise, this field is initialized as zero of bytes32.

See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol#L84>

### The bug

In the function `mintingPaymentDefault` (<https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/CollateralReservations.sol#L144-L179>), the verification logic can be split into two parts:

1. **Source address validation:**

```solidity
require(!_nonPayment.data.requestBody.checkSourceAddresses && crt.sourceAddressesRoot == bytes32(0) ||
        _nonPayment.data.requestBody.checkSourceAddresses &&
        crt.sourceAddressesRoot == _nonPayment.data.requestBody.sourceAddressesRoot,
        "invalid check or source addresses root");
```

2. **Criteria matching between request and response:**

```solidity
require(_nonPayment.data.requestBody.standardPaymentReference == PaymentReference.minting(_crtId) &&
        _nonPayment.data.requestBody.destinationAddressHash == agent.underlyingAddressHash &&
        _nonPayment.data.requestBody.amount == underlyingValueUBA + crt.underlyingFeeUBA,
        "minting non-payment mismatch");

require(_nonPayment.data.responseBody.firstOverflowBlockNumber > crt.lastUnderlyingBlock &&
        _nonPayment.data.responseBody.firstOverflowBlockTimestamp > crt.lastUnderlyingTimestamp,
        "minting default too early");

require(_nonPayment.data.requestBody.minimalBlockNumber <= crt.firstUnderlyingBlock,
        "minting non-payment proof window too short");
```

The first check allows two scenarios:

* If `checkSourceAddresses` is false, `crt.sourceAddressesRoot` must be `bytes32(0)`, indicating a reservation without handshake.
* If `checkSourceAddresses` is true, the provided `sourceAddressesRoot` must match `crt.sourceAddressesRoot`.

This logic is flawed and can be best demonstrate with the following scenario:

* A user creates a collateral reservation without a handshake, so `crt.sourceAddressesRoot` is set to `bytes32(0)`.
* The user makes a valid payment but doesn’t call `executeMinting` in time, expecting a designated executor to do so.
* A non-payment proof request should now fail, since a matching transaction exists.
* However, a malicious agent can craft a non-payment proof request with `checkSourceAddresses = true` and set `sourceAddressesRoot = bytes32(0)`.
* This causes the FDC client to search for a payment from `bytes32(0)`, which doesn't exist, and thus generates a valid-looking non-payment proof.
* When the agent submits this proof to `mintingPaymentDefault`, it passes the source address check since `crt.sourceAddressesRoot == bytes32(0)`.

**References for logic on FDC**

* <https://github.com/flare-foundation/verifier-indexer-api/blob/main/src/verification/referenced-payment-nonexistence/referenced-payment-nonexistence.ts#L196-L206>
* <https://github.com/flare-foundation/verifier-indexer-api/blob/main/src/indexed-query-manager/XrpIndexerQueryManager.ts#L84-L88>

As a result, a malicious agent can fraudulently trigger a minting default, despite a valid payment being made, and retain both underlying asset and the reservation fee.

### The possibiilty

The likelihood of exploitation is non-zero. Users typically delegate executeMinting to automation bots by specifying them as executors in reserveCollateral, and do not submit the payment proof themselves.

If the executor bot fails — due to malfunction, lack of funds, or downtime — a valid payment may remain unproven. This risk increases when users make the payment near the end of the allowed window (e.g., at minute 14 of a 15-minute window), which can happen due to network delays on the underlying chain.

In such cases, a malicious agent can exploit the delay by submitting a forged non-payment proof and triggering a false minting default.

## Impact

* **Loss of Funds**: While this may be an edge case and only executable by agents, exploitation results in a direct loss of funds — the attacker can retain both the underlying collateral and the reservation fee, despite a valid payment being made by the user.
* **Severity: Medium to High**: The impact is significant and could be considered critical. However, given the limited scope (only agents can exploit it and specific conditions are required), a downgrade to medium or high severity may be justified.

## Recommended Mitigations

```solidity
require(!_nonPayment.data.requestBody.checkSourceAddresses && crt.sourceAddressesRoot == bytes32(0) ||
        _nonPayment.data.requestBody.checkSourceAddresses &&
        crt.sourceAddressesRoot == _nonPayment.data.requestBody.sourceAddressesRoot &&
        crt.sourceAddressesRoot != bytes32(0),
        "invalid check or source addresses root");
```

## Proof of Concept

## Proof-of-Concept

Given that the mock FDC in the test suite has the exact same logic as the actual FDC client, the proof-of-concept is created to demonstrate that a proof of non-existence payment can be generated with `checkSourceAddresses = true` and `sourceAddresses = bytes32(0)` and also show that it can be used to fraudently call `mintingPaymentDefault` despite the fact that there is a valid payment.

1. Add the following test in: `test/integration/fasset-simulation/03-MintingFailures.ts`

```typescript
    it("nnez - forced a default on minter", async () => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000));
        // make agent available
        const fullAgentCollateral = toWei(3e8);
        await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
        // update block
        await context.updateUnderlyingBlock();
        
        // Reserve collateral, non-handshake type  
        const lots = 3;
        const crt = await minter.reserveCollateral(agent.vaultAddress, lots);
        
        // Perform valid payment within time window  
        // minter.performMintingPayment always perform a valid payment  
        const txHash = await minter.performMintingPayment(crt);
        // Mine some blocks
        for (let i = 0; i <= context.chainInfo.underlyingBlocksForPayment + 10; i++) {
            await minter.wallet.addTransaction(minter.underlyingAddress, minter.underlyingAddress, 1, null);
        }
        
        // Time has now passed beyond payment window  
        // Request a proof of non-payment existence, but specify the source as bytes32(0)  
        const proof = await context.attestationProvider.proveReferencedPaymentNonexistence(
            agent.underlyingAddress,
            crt.paymentReference,
            crt.valueUBA.add(crt.feeUBA),
            crt.firstUnderlyingBlock.toNumber(),
            crt.lastUnderlyingBlock.toNumber(),
            crt.lastUnderlyingTimestamp.toNumber(),
            "0x0000000000000000000000000000000000000000000000000000000000000000");  
        // The proof is generated, and indication that FDC doesn't find matching transaction despite the fact that there is one  
        // Malicious agent can invoke `mintingPaymentDefault`  
        const res = await context.assetManager.mintingPaymentDefault(proof, crt.collateralReservationId, { from: agent.ownerWorkAddress });
        // Observe in the log that the execution is success, minting default event is emitted  
        console.log(res);
    });
```

2. Run `yarn hardhat test "test/integration/fasset-simulation/03-MintingFailures.ts" --grep "nnez - forced a default on minter"`
