# #38225 \[SC-Insight] user funds will get stuck if \`removeDestination\` executes before notarization an

**Submitted on Dec 28th 2024 at 08:15:12 UTC by @OxAnmol for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38225
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/Bridge.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 30 days
  * Permanent freezing of funds

## Description

## Brief/Intro

If `Bridge:removeDestination` is called before `Bridge:authNotary` or `Bridge:withdraw`, and the removed destination is the user's source chain, subsequent transactions to notarise will revert—causing user funds to become stuck.

## Vulnerability Details

Admins have the ability to remove configured destinations at any time. If a user has recently deposited from a chain that is about to be removed and their deposit is in the `consortium` for validation, a subsequent `removeDestination` call can occur without anyone's knowledge. When the user's deposit is later validated and the consortium attempts to notarize the deposit, the transaction will revert due to this check:

```solidity
 function authNotary(
        bytes calldata payload,
        bytes calldata proof
    ) external nonReentrant {
        ...SKIP...
 
        // Ensure that fromContract matches the bridgeContract
        // This make sure that call is not comming from unsupported bridge contract
        DestinationConfig memory destConf = getDestination(
            bytes32(action.fromChain)
        );

        if (destConf.bridgeContract != action.fromContract) {
@>            revert UnknownOriginContract(
                bytes32(action.fromChain),
                action.fromContract
            );
        }

        ...SKIP...
    }
```

### Example Scenario

1. User calls `BridgeInBsc:deposit` with 1 LBTC to bridge from BSC to Base.
2. Admin calls `Bridge:removeDestination` in Base to remove support for BSC to Base bridging.
3. User's 1 LBTC is burned in BSC chain and their deposit enters consortium validation.
4. While the deposit is in consortium validation, `removeDestination` executes and the Base→BSC bridge is removed.
5. When `authNotary` is called after deposit validation, the transaction reverts because the source chain (BSC) is no longer supported, leaving the user's 1 LBTC stuck until BSC support is restored.

## Impact Details

User funds will remain stuck until support for the removed destination is added again. I believe this is a temporary.

According to the immunefi severity guide this should be high because it is Temporary freezing of funds

## References

<https://github.com/lombard-finance/evm-smart-contracts/blob/a818ea0489178ccd00019edab24637c38501af7b/contracts/bridge/Bridge.sol#L237>

## Proof of Concept

## Proof of Concept

Paste this test in `Bridge.ts` → `Actions/Flows`

```js
it('should demonstrate funds getting stuck when destination removed during consortium validation', async function () {
            const amount = AMOUNT;
            const fee = amount / 10n;
            const amountWithoutFee = amount - fee;
            const receiver = signer2.address;

            // 1. First approve and deposit LBTC
            await lbtcSource
                .connect(signer1)
                .approve(await bridgeSource.getAddress(), amount);

            const tx = await bridgeSource
                .connect(signer1)
                .deposit(CHAIN_ID, encode(['address'], [receiver]), amount);

            // Get the deposit payload from the event
            const receipt = await tx.wait();

            // 2. Admin removes the destination before consortium validation
            await bridgeDestination.removeDestination(CHAIN_ID);

            // 3. Get consortium signature
            const data = await signDepositBridgePayload(
                [signer1],
                [true],
                CHAIN_ID,
                await bridgeSource.getAddress(),
                CHAIN_ID,
                await bridgeDestination.getAddress(),
                receiver,
                amountWithoutFee
            );

            // 4. Try to notarize - this should fail
            await expect(
                bridgeDestination
                    .connect(signer1)
                    .authNotary(data.payload, data.proof)
            ).to.be.revertedWithCustomError(
                bridgeDestination,
                'UnknownOriginContract'
            );

            // 5. Verify funds are effectively stuck
            expect(await lbtcSource.balanceOf(signer1.address)).to.equal(0); // User lost their funds
            expect(await lbtcDestination.balanceOf(receiver)).to.equal(0); // Receiver never got the funds
        }); 

```
