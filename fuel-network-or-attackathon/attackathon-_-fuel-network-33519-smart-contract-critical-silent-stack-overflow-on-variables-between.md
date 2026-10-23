# Attackathon \_ Fuel Network 33519 - \[Smart Contract - Critical] Silent Stack overflow on variables be

Submitted on Mon Jul 22 2024 07:57:32 GMT-0400 (Atlantic Standard Time) by @Minato7namikazi for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33519

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

There is Silent Stack overflow on variables between cross-contract calls can happen

## Vulnerability Details

In large codebases and within complex functions that have many cross-contract calls u256 variables can overflow and cause critical damages

Using log , I found that happens more than once:

```
before: variable: 6000000000000000000000000000000000 (u256)

after: variable: 294731856024973518640372915683249701534862079315
```

## Impact Details

* When a stack overflow occurs silently, it can overwrite adjacent memory locations without raising an error. In the context of cross-contract calls, this could lead to corrupted state variables or parameters being passed between contracts.
* Corrupted data could lead to significant financial losses. Incorrect balances, misrouted transactions, faulty trade executions could result in substantial $$$ damage.

## Regarding the PoC

this happened while playing with large sway codebase for an exchange project .. i'm literally submitting this at the last minute of the attackathon : D .. so this report will be continued with another minimized PoC in the comments section

## Proof of concept

#### `https://github.com/minato7namikazi/ruscet-contracts`

Compile and run the `PositionRouter.test.ts`

```
    it("increasePosition acceptablePrice short", async () => {
        await USDC.functions.mint(contrToAccount(vault), expandDecimals(8000)).call()
        await vault.functions.buy_rusd(toAsset(USDC), addrToAccount(user1)).addContracts(attachedContracts).call()

        await router.as(user0).functions.set_approved_plugins(toContract(positionRouterBPM), true).call()

        await BNB.functions.mint(addrToAccount(user0), expandDecimals(2)).call()
        await BNB.functions.mint(addrToAccount(deployer), expandDecimals(2)).call()

        const amountIn = expandDecimals(2)

        await expect(
            positionRouter
                .as(user0)
                .multiCall([
                    utils.functions.transfer_assets_to_contract(toAsset(BNB), amountIn, toContract(positionRouter)).callParams({
                        forward: [amountIn, getAssetId(BNB)],
                    }),
                    positionRouter.functions
                        .increase_position(
                            [toAsset(BNB), toAsset(USDC)], // path
                            toAsset(BNB), // index_asset
                            amountIn, // amountIn
                            expandDecimals(1), // minOut
                            toUsd(6000), // size_delta
                            false, // is_long
                            toUsd(310), // acceptablePrice
                            referralCode, // referralCode
                        )
                        .addContracts(attachedContracts),
                ])
                .call(),
        ).to.be.revertedWith("BPMMarkPriceLtPrice")
    })

    it("maxGlobalShortSize", async () => {
        await USDC.functions.mint(contrToAccount(vault), expandDecimals(8000)).call()
        await vault.functions.buy_rusd(toAsset(USDC), addrToAccount(user1)).addContracts(attachedContracts).call()

        await positionRouterBPM.functions
            .set_max_global_sizes([toAsset(BNB), toAsset(USDC)], [0, 0], [toUsd(5000), toUsd(10000)])
            .addContracts(attachedContracts)
            .call()

        await router.as(user0).functions.set_approved_plugins(toContract(positionRouterBPM), true).call()

        await BNB.functions.mint(addrToAccount(user0), expandDecimals(2)).call()
        await BNB.functions.mint(addrToAccount(deployer), expandDecimals(2)).call()

        const amountIn = expandDecimals(2)

        const tx = positionRouter.as(user0).multiCall([
            utils.functions.transfer_assets_to_contract(toAsset(BNB), amountIn, toContract(positionRouter)).callParams({
                forward: [amountIn, getAssetId(BNB)],
            }),
            positionRouter.functions
                .increase_position(
                    [toAsset(BNB), toAsset(USDC)], // path
                    toAsset(BNB), // index_asset
                    amountIn, // amountIn
                    expandDecimals(1), // minOut
                    toUsd(6000), // size_delta
                    false, // is_long
                    toUsd(290), // acceptablePrice
                    referralCode, // referralCode
                )
                .addContracts(attachedContracts),
        ])
        await expect(tx.call()).to.be.revertedWith("BPMMaxShortsExceeded")

        await positionRouterBPM.functions
            .set_max_global_sizes([toAsset(BNB), toAsset(USDC)], [0, 0], [toUsd(6000), toUsd(10000)])
            .addContracts(attachedContracts)
            .call()

        expect(await getValStr(vaultUtils.functions.get_global_short_sizes(toAsset(BNB)))).eq("0")
        await tx.call()
        expect(await getValStr(vaultUtils.functions.get_global_short_sizes(toAsset(BNB)))).eq(
            "6000000000000000000000000000000000",
        )
    })
```
