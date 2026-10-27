# #47082 \[SC-Low] Zero collateral payout despite burned fAssets

**Submitted on Jun 8th 2025 at 19:25:48 UTC by @dldLambda for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47082
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/RedemptionRequests.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

There is a vulnerability in the collateral redemption logic where the conversion of AMG (a unit representing fractional assets) to token wei (native token units) suffers from precision loss due to scaling and rounding. As a result, under certain conditions, redeemers burn their fAssets but receive zero collateral payout. This issue can lead to users losing expected payouts, undermining trust and usability.

## Vulnerability Details

Vulnerability manifests itself in function "redeemFromAgentInCollateral".

Call chain:

redeemFromAgentInCollateral (RedemptionRequestsFacet.sol) ---> RedemptionRequests.redeemFromAgentInCollateral

```
    function redeemFromAgentInCollateral(
        address _agentVault,
        address _redeemer,
        uint256 _amountUBA
    )
        internal
    {
        Agent.State storage agent = Agent.get(_agentVault);
        Agents.requireCollateralPool(agent);
        require(_amountUBA != 0, "redemption of 0");
        // close redemption tickets
        uint64 amountAMG = Conversion.convertUBAToAmg(_amountUBA);
        (uint64 closedAMG, uint256 closedUBA) = Redemptions.closeTickets(agent, amountAMG, true, false);
        // pay in collateral
        uint256 priceAmgToWei = Conversion.currentAmgPriceInTokenWei(agent.vaultCollateralIndex);
        uint256 paymentWei = Conversion.convertAmgToTokenWei(closedAMG, priceAmgToWei)
            .mulBips(agent.buyFAssetByAgentFactorBIPS);
        Agents.payoutFromVault(agent, _redeemer, paymentWei);
        emit IAssetManagerEvents.RedeemedInCollateral(_agentVault, _redeemer, closedUBA, paymentWei);
        // burn the closed assets
        Redemptions.burnFAssets(msg.sender, closedUBA);
    }
```

What happens in the logic:

1. The user (redeemer) owns some fAssets — tokens that represent a share in the collateral pool.
2. The user calls the fAssets redemption function to burn their fAssets and get back the collateral — a native token (e.g. USDC, FLR, etc.).
3. The payout amount is calculated using the AMG price (the conventional unit associated with fAssets) in units of the collateral token (tokenWei).
4. The AMG price is taken from the oracle (FTSO), then multiplied by the amount of AMG being burned and divided by the scaling factor to get the amount of tokens to be paid out.

However, under certain conditions, the function "convertAmgToTokenWei" can round the value to zero, which leads to a zero payout when burning the user's fassets:

```
    function convertAmgToTokenWei(uint256 _valueAMG, uint256 _amgToTokenWeiPrice) internal pure returns (uint256) {
        return _valueAMG.mulDiv(_amgToTokenWeiPrice, AMG_TOKEN_WEI_PRICE_SCALE);
    }
```

For this, a very low value of "\_amgToTokenWeiPrice" is required - which is possible with specific tokens or/and a reduced price of AMG (and this does not depend on the decimals) - which is theoretically possible, since when obtaining this value, an FTSO oracle is used, which, like any other, can be subject to fluctuations due to high volatility:

```
uint256 priceAmgToWei = Conversion.currentAmgPriceInTokenWei(agent.vaultCollateralIndex);
```

```
    function currentAmgPriceInTokenWei(
        uint256 _tokenType
    )
        internal view
        returns (uint256 _price)
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        (_price,,) = currentAmgPriceInTokenWeiWithTs(state.collateralTokens[_tokenType], false);
    }
```

```
    function currentAmgPriceInTokenWeiWithTs(
        CollateralTypeInt.Data storage _token,
        bool _fromTrustedProviders
    )
        internal view
        returns (uint256 /*_price*/, uint256 /*_assetTimestamp*/, uint256 /*_tokenTimestamp*/)
    {
        (uint256 assetPrice, uint256 assetTs, uint256 assetFtsoDec) =
            readFtsoPrice(_token.assetFtsoSymbol, _fromTrustedProviders);
        if (_token.directPricePair) {
            uint256 price = calcAmgToTokenWeiPrice(_token.decimals, 1, 0, assetPrice, assetFtsoDec);
            return (price, assetTs, assetTs);
        } else {
            (uint256 tokenPrice, uint256 tokenTs, uint256 tokenFtsoDec) =
                readFtsoPrice(_token.tokenFtsoSymbol, _fromTrustedProviders);
            uint256 price =
                calcAmgToTokenWeiPrice(_token.decimals, tokenPrice, tokenFtsoDec, assetPrice, assetFtsoDec);
            return (price, assetTs, tokenTs);
        }
    }
```

```
    function calcAmgToTokenWeiPrice(
        uint256 _tokenDecimals,
        uint256 _tokenPrice,
        uint256 _tokenFtsoDecimals,
        uint256 _assetPrice,
        uint256 _assetFtsoDecimals
    )
        internal view
        returns (uint256)
    {
        AssetManagerSettings.Data storage settings = Globals.getSettings();
        uint256 expPlus = _tokenDecimals + _tokenFtsoDecimals + AMG_TOKEN_WEI_PRICE_SCALE_EXP;
        uint256 expMinus = settings.assetMintingDecimals + _assetFtsoDecimals;
        // If negative, price would probably always be 0 after division, so this is forbidden.
        // Anyway, we should know about this before we add the token and/or asset, since
        // token decimals and ftso decimals typically never change.
        assert(expPlus >= expMinus);
        return _assetPrice.mulDiv(10 ** (expPlus - expMinus), _tokenPrice);
    }
```

(It is noteworthy that this situation is possible even if function "calcAmgToTokenWeiPrice" returns a normal value)

If \_valueAMG\*\_amgToTokenWeiPrice < AMG\_TOKEN\_WEI\_PRICE\_SCALE - rounding down to zero occurs and the user receives nothing, but his fassets tokens are still burned.

This situation is possible, since the value of variable "\_amgToTokenWeiPrice" depends on the price received from the oracle - and is possible even with other normally detailed parameters. This market price may be temporary - however, users irrevocably lose their funds.

## Impact Details

Users redeeming fAssets can receive zero payout, losing their expected collateral returns.

This loss is due to rounding and not due to protocol insolvency or direct theft.

The protocol retains collateral; funds are not stolen or lost, but payouts are unfairly zeroed out.

This may erode user trust and negatively impact protocol usability.

Exploiting this requires an external condition (AMG price dropping significantly) which depends on oracle data and market dynamics.

The bug does not cause loss of collateral but violates promised returns.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/RedemptionRequests.sol#L97> - function redeemFromAgentInCollateral

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Conversion.sol#L182> - function convertAmgToTokenWei

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Conversion.sol#L162> - function calcAmgToTokenWeiPrice

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Conversion.sol#L128> - function currentAmgPriceInTokenWeiWithTs

## Proof of Concept

## Proof of Concept

1. change value 1000000000000 ---> 1000000000 (I tested it on smaller value) (<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/test/unit/fasset/library/Redemption.ts#L167>)
2. add this import (import "hardhat/console.sol";) in RedemptionRequests.sol (<https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/RedemptionRequests.sol#L15>) - to use console.log()
3. add a modifications to function redeemFromAgentInCollateral (<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/RedemptionRequests.sol#L97>)

//modify1-2,5-7 to clearly see all the values in the logs

//modify2-3 - to simulate significant change in price

```
    function redeemFromAgentInCollateral(
        address _agentVault,
        address _redeemer,
        uint256 _amountUBA
    )
        internal
    {
        Agent.State storage agent = Agent.get(_agentVault);
        Agents.requireCollateralPool(agent);
        require(_amountUBA != 0, "redemption of 0");
        // close redemption tickets
        uint64 amountAMG = Conversion.convertUBAToAmg(_amountUBA);
        console.log("_amountUBA", _amountUBA); //modify1
        console.log("amountAMG", amountAMG); //modify2
        (uint64 closedAMG, uint256 closedUBA) = Redemptions.closeTickets(agent, amountAMG, true, false);
        // pay in collateral
        //uint256 priceAmgToWei = Conversion.currentAmgPriceInTokenWei(agent.vaultCollateralIndex); //modify3
        uint256 priceAmgToWei = 1000000000; //modify4
        console.log("agent.buyFAssetByAgentFactorBIPS", agent.buyFAssetByAgentFactorBIPS);
        uint256 paymentWei = Conversion.convertAmgToTokenWei(closedAMG, priceAmgToWei)
            .mulBips(agent.buyFAssetByAgentFactorBIPS);

        console.log("closedAMG", closedAMG); //modify5
        console.log("priceAmgToWei",priceAmgToWei); //modify6
        console.log("paymentWei", paymentWei); //modify7
        Agents.payoutFromVault(agent, _redeemer, paymentWei);
        emit IAssetManagerEvents.RedeemedInCollateral(_agentVault, _redeemer, closedUBA, paymentWei);
        // burn the closed assets
        Redemptions.burnFAssets(msg.sender, closedUBA);
    }
```

4. Add into test/unit/fasset/library/Redemption.ts and run this test with the command (yarn testHH --grep "dl\_dLambda\_test3"):

```
    it("dl_dLambda_test3", async () => {
        // init
        const agentVault = await createAgent(agentOwner1, underlyingAgent1);

        await depositAndMakeAgentAvailable(agentVault, agentOwner1);
        collateralPool = await CollateralPool.at(await assetManager.getCollateralPool(agentVault.address));
        const vaultCollateralBalanceAgentBefore = await usdc.balanceOf(agentVault.address);
        const vaultCollateralBalanceRedeemerBefore = await usdc.balanceOf(redeemerAddress1);

        await mintAndRedeemFromAgentInCollateral(agentVault, collateralPool.address, chain, underlyingMinter1, minterAddress1, redeemerAddress1, true);
        const amountUBA = toBN(settings.assetMintingGranularityUBA).div(toBN(1e9)); // 1 AMG

        //check vault collateral balances
        const vaultCollateralBalanceAgentAfter = await usdc.balanceOf(agentVault.address);
        const vaultCollateralBalanceRedeemerAfter = await usdc.balanceOf(redeemerAddress1);
     assert.equal(vaultCollateralBalanceAgentBefore.sub(vaultCollateralBalanceAgentAfter).toString(), vaultCollateralBalanceRedeemerAfter.sub(vaultCollateralBalanceRedeemerBefore).toString())
    });
```

5. And you will see the following output in the logs:

```
  Contract: Redemption.sol; test/unit/fasset/library/Redemption.ts; Redemption basic tests
_amountUBA 1000000000
amountAMG 1
agent.buyFAssetByAgentFactorBIPS 9000
closedAMG 1
priceAmgToWei 1000000000
paymentWei 0
    ✔ dl_dLambda_test3 (156ms)


  1 passing (6s)

✨  Done in 22.68s.
```

6. closedAMG - is how much will be burned. paymentWei - is how much will be paid to the user.
