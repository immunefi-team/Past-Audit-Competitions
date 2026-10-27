# #38231 \[SC-Low] Due to incorrect design in \`Consortium::setNextValidatorSet\` the validator set could

**Submitted on Dec 28th 2024 at 14:51:40 UTC by @MrMorningstar for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38231
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/consortium/Consortium.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol (not lower than $1K))

## Description

## Brief/Intro

After the initial validator is set by owner in `setInitialValidatorSet` , the new validator set can be set via `setNextValidatorSet` which looks like this:

```js
    function setNextValidatorSet(
        bytes calldata payload,
        bytes calldata proof
    ) external {
        // payload validation
        if (bytes4(payload) != Actions.NEW_VALSET) {
            revert UnexpectedAction(bytes4(payload));
        }
        Actions.ValSetAction memory action = Actions.validateValSet(
            payload[4:]
        );

        ConsortiumStorage storage $ = _getConsortiumStorage();

        // check proof
        bytes32 payloadHash = sha256(payload);
        checkProof(payloadHash, proof);

        if (action.epoch != $.epoch + 1) revert InvalidEpoch();

        _setValidatorSet( // qanswered check this func and input params
            $,
            action.validators,
            action.weights,
            action.weightThreshold,
            action.epoch
        );
    }
```

Anyone can call the `setNextValidatorSet` but it requires signatures by the current consortium to be valid.

## Vulnerability Details

The new validator set will typically be set every epoch, but there is no obligation to that which means there is possibility that one validator set could be valid for multiple epochs.

The issue arise with this check:

```js
if (action.epoch != $.epoch + 1) revert InvalidEpoch();
```

This check expect that new set is set every epoch, which means if one set was valid for multiple epochs and it was not changed, it will not be possible to set a new validator set .

Example:

If last validator set is set in epoch 10 and it was valid for 2 epochs so the attempt is to set a new validator set in epoch 12, the function would not allow that.

This is demonstrated in the PoC provided.

## Impact Details

New validator set could not be set. Which could cause issues to the protocol and its users (especially if some validators became malleable after some time and misbehave).

## Recommendation

Make the following change in `setNextValidatorSet`:

```diff
    function setNextValidatorSet(
        bytes calldata payload,
        bytes calldata proof
    ) external {
        // payload validation
        if (bytes4(payload) != Actions.NEW_VALSET) {
            revert UnexpectedAction(bytes4(payload));
        }
        Actions.ValSetAction memory action = Actions.validateValSet(
            payload[4:]
        );

        ConsortiumStorage storage $ = _getConsortiumStorage();

        // check proof
        bytes32 payloadHash = sha256(payload);
        checkProof(payloadHash, proof);

-        if (action.epoch != $.epoch + 1) revert InvalidEpoch();
+        if (action.epoch < $.epoch + 1) revert InvalidEpoch();

        _setValidatorSet( // qanswered check this func and input params
            $,
            action.validators,
            action.weights,
            action.weightThreshold,
            action.epoch
        );
    }
```

## Proof of Concept

## Proof of Concept

Paste this test in `Consortium.ts` :

```js
        it('cannot set next set if previous one was valid for multiple epochs', async function () {
            const data = await signNewValSetPayload(
                [signer3, signer1, signer2],
                [true, true, false],
                12, // epoch 12
                [signer1.publicKey, signer2.publicKey],
                [1, 2],
                3,
                1
            );
            await expect(lombard.setNextValidatorSet(data.payload, data.proof))
                .to.revertedWithCustomError(lombard,"InvalidEpoch");
        });
```

Execute the following command:

```
yarn hardhat test --grep "cannot set next set if previous one was valid for multiple epochs"
```

The test will pass which will prove the flaw in the design

```
  Consortium
    With Initial ValidatorSet
      ✔ cannot set next set if previous one was valid for multiple epochs (168ms)


  1 passing (2s)
```
