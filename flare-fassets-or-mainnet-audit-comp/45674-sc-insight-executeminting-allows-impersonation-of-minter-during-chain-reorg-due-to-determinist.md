# #45674 \[SC-Insight] \`executeMinting()\` allows impersonation of minter during chain-reorg due to deterministic \`crtId\` and lack of minter binding

**Submitted on May 18th 2025 at 23:06:32 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45674
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

An attacker can front-run a legitimate collateral reservation during a reorg and copy the `crtId` by ensuring their reservation gets the same `randomizedIdSkip()` result. By setting the executor field in their reservation to the real user’s executor and other data, they create a situation where the real executor, when executing with a valid payment proof, unintentionally mints tokens for the attacker’s address.

```solidity
   function reserveCollateral(
        address _minter,
        address _agentVault,
        uint64 _lots,
        uint64 _maxMintingFeeBIPS,
        address payable _executor,
        string[] calldata _minterUnderlyingAddresses
    )
        internal
    {
        Agent.State storage agent = Agent.get(_agentVault);
        Agents.requireWhitelistedAgentVaultOwner(agent);

        Collateral.CombinedData memory collateralData = AgentCollateral.combinedData(agent);
        AssetManagerState.State storage state = AssetManagerState.get();

...............

        require(msg.value >= reservationFee, "inappropriate fee amount");
        // create new crt id - pre-increment, so that id can never be 0
       @>> state.newCrtId += PaymentReference.randomizedIdSkip();
        uint64 crtId = state.newCrtId;
        // create in-memory cr and then put it to storage to not go out-of-stack
        CollateralReservation.Data memory cr;
        cr.valueAMG = valueAMG;
        cr.underlyingFeeUBA = Conversion.convertAmgToUBA(valueAMG).mulBips(agent.feeBIPS).toUint128();
        cr.reservationFeeNatWei = reservationFee.toUint128();
        // 1 is added for backward compatibility where 0 means "value not stored" - it is subtracted when used
        cr.poolFeeShareBIPS = agent.poolFeeShareBIPS + 1;
        cr.agentVault = _agentVault;
        cr.minter = _minter;
        cr.executor = _executor;
        cr.executorFeeNatGWei = ((msg.value - reservationFee) / Conversion.GWEI).toUint64();

........
         }
```

The `reserveCollateral()` flow assigns a `crtId` using a weak randomization mechanism:

```solidity
function randomizedIdSkip() internal view returns (uint64) {
        // This is rather weak randomization, but it's ok for the purpose of preventing speculative underlying
        // payments, since there is only one guess possible - the first mistake makes agent liquidated.
        //slither-disable-next-line weak-prng
      @>>   return uint64(block.number % ID_RANDOMIZATION + 1);
    }

```

Since this logic is block-dependent and lacks any entropy from the caller, it is predictable during reorgs. This enables an attacker to force a collision with a legitimate `crtId` reservation.

Because `executeMinting()` trusts that the caller (minter, executor, or agent) corresponds to the original reservation, and does not validate the minter address against an expected value, the following sequence is possible:

1. Legitimate user submits a reservation.
2. Attacker front-runs in a reorged block and creates their own reservation with same `crtId` (by controlling timing).
3. Attacker sets executor to the known executor used by the real user.
4. Real executor (most likely an offchain relayer) calls `executeMinting()` believing they're minting for the legit user, but ends up minting to attacker-controlled address.

```solidity
function executeMinting(
        IPayment.Proof calldata _payment,
        uint64 _crtId
    )
        internal
    {
        CollateralReservation.Data storage crt = CollateralReservations.getCollateralReservation(_crtId);
        require(crt.handshakeStartTimestamp == 0, "collateral reservation not approved");
        Agent.State storage agent = Agent.get(crt.agentVault);
        // verify transaction
        TransactionAttestation.verifyPaymentSuccess(_payment);
        // minter or agent can present the proof - agent may do it to unlock the collateral if minter
        // becomes unresponsive
       @>> require(msg.sender == crt.minter || msg.sender == crt.executor || Agents.isOwner(agent, msg.sender),
            "only minter, executor or agent");
        require(_payment.data.responseBody.standardPaymentReference == PaymentReference.minting(_crtId),
            "invalid minting reference");
        require(_payment.data.responseBody.receivingAddressHash == agent.underlyingAddressHash,
            "not minting agent's address");
        require(crt.sourceAddressesRoot == bytes32(0) ||
                crt.sourceAddressesRoot == _payment.data.responseBody.sourceAddressesRoot, // handshake was required
            "invalid minter underlying addresses root");

        uint256 mintValueUBA = Conversion.convertAmgToUBA(crt.valueAMG);
        require(_payment.data.responseBody.receivedAmount >= SafeCast.toInt256(mintValueUBA + crt.underlyingFeeUBA),
            "minting payment too small");
        // we do not allow payments before the underlying block at requests, because the payer should have guessed
        // the payment reference, which is good for nothing except attack attempts
        require(_payment.data.responseBody.blockNumber >= crt.firstUnderlyingBlock,
            "minting payment too old");
        // mark payment used
        AssetManagerState.get().paymentConfirmations.confirmIncomingPayment(_payment);
        // execute minting
        _performMinting(agent, MintingType.PUBLIC, _crtId, crt.minter, crt.valueAMG,
            uint256(_payment.data.responseBody.receivedAmount), calculatePoolFeeUBA(agent, crt));
        // pay to executor if they called this method
        uint256 unclaimedExecutorFee = crt.executorFeeNatGWei * Conversion.GWEI;

        if (msg.sender == crt.executor) {
            // safe - 1) guarded by nonReentrant in AssetManager.executeMinting, 2) recipient is msg.sender
            Transfers.transferNAT(crt.executor, unclaimedExecutorFee);
            unclaimedExecutorFee = 0;
        }

        // pay the collateral reservation fee (guarded against reentrancy in AssetManager.executeMinting)
        CollateralReservations.distributeCollateralReservationFee(agent,
            crt.reservationFeeNatWei + unclaimedExecutorFee);
        // cleanup
        CollateralReservations.releaseCollateralReservation(crt, _crtId);   // crt can't be used after this
    }
```

## Impact

Likelihood: Low – reorgs are rare, but deterministic ID generation enables predictability.

Impact: High - The legitimate minter loses funds permanently. The attacker gains tokens minted using their own reservation.

## Recommendation

To prevent this attack, bind `executeMinting()` to a validated minter identity:

1. Add an `expectedMinter` parameter to `executeMinting()`.
2. Add `require(crt.minter == expectedMinter)` to validate against impersonation.

This ensures the executor cannot be tricked into executing a minting for a spoofed reservation.

```solidity
require(crt.minter == expectedMinter, "minter mismatch");
```

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/CollateralReservationsFacet.sol#L36-L51>

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/MintingFacet.sol#L24-L32>

## Proof of Concept

## Proof of Concept

The attacker can follow this part:

1. Legitimate user submits a reservation.
2. Attacker front-runs in a reorged block and creates their own reservation with same `crtId` (by controlling timing).
3. Attacker sets executor to the known executor used by the real user.
4. Real executor (most likely an offchain relayer) calls `executeMinting()` believing they're minting for the legit user, but ends up minting to attacker-controlled address.
