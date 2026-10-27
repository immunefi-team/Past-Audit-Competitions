# #38644 \[SC-Insight] Q\&A

**Submitted on Jan 8th 2025 at 15:45:28 UTC by @focusoor for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38644
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## \[N-1] Unused `payload`field inside CLAdapter:\_buildCCIPMessage

The `_buildCCIPMessage` method is used to construct a CCIP message for bridging LBTC tokens. When constructing token bridging without an additional call, the data field is not required and will be set to empty. Therefore, the `_payload` parameter in the function is unused and can be removed. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L225>

```solidity
 function _buildCCIPMessage(
        bytes memory _receiver,
        uint256 _amount,
->    bytes memory _payload
    ) private view returns (Client.EVM2AnyMessage memory) {
...
return
            Client.EVM2AnyMessage({
                receiver: _receiver,
->            data: "",
                tokenAmounts: tokenAmounts,
                extraArgs: Client._argsToBytes(
                    Client.EVMExtraArgsV2({
                        gasLimit: getExecutionGasLimit,
                        allowOutOfOrderExecution: true
                    })
                ),
                feeToken: address(0) // let's pay with native tokens
            });
```

Additionally, the `CLAdapter:getFee` method can be modified to remove the `_payload` parameter, as it is not required in the context of token bridging without an additional call. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L92>

```solidity
function getFee(
        bytes32 _toChain,
        bytes32,
        bytes32 _toAddress,
        uint256 _amount,
->    bytes memory _payload
    ) public view override returns (uint256) {
        return
            IRouterClient(tokenPool.getRouter()).getFee(
                getRemoteChainSelector[_toChain],
                _buildCCIPMessage(
                    abi.encodePacked(_toAddress),
                    _amount,
->                _payload
                )
            );
    }
```

Furthermore, inside the `Bridge:getAdapterFee` method, the length of the payload data is not relevant, and passing `new bytes(228)` is unnecessary. This parameter can be safely removed to simplify the implementation. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L133>

```solidity
function getAdapterFee(
        bytes32 toChain,
        bytes32 toAddress,
        uint64 amount
    ) external view returns (uint256) {
        DestinationConfig memory destConfig = getDestination(toChain);
        if (destConfig.bridgeContract == bytes32(0)) {
            return 0;
        }
-      // payload data doesn't matter for fee calculation, only length
        return
            destConfig.adapter.getFee(
                toChain,
                destConfig.bridgeContract,
                toAddress,
                amount,
-              new bytes(228)
            );
    }
```

## \[N-2] `CLAdapter:initWithdrawalNoSignatures` is not used.

The `CLAdapter:initWithdrawalNoSignatures` method is not used in the current implementation and can be safely removed. Consider adding this method back only if unnotarized payloads are ever permitted in the future. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L192>

## \[N-3] Unnecessary storage read inside `LBTC:batchMintWithFee`

Inside the `batchMintWithFee` method, before calling `_mintWithFee`, there is an unnecessary call to `_getLBTCStorage` that is not used. Consider removing it to save gas costs and improve code clarity. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/LBTC/LBTC.sol#L446>

```solidity
function batchMintWithFee(
        bytes[] calldata mintPayload,
        bytes[] calldata proof,
        bytes[] calldata feePayload,
        bytes[] calldata userSignature
    ) external onlyClaimer {
        ...
-      LBTCStorage storage $ = _getLBTCStorage();
        for (uint256 i; i < mintPayload.length; ++i) {
            _mintWithFee(
                mintPayload[i],
                proof[i],
                feePayload[i],
                userSignature[i]
            );
        }
    }
```

## \[N-4] No constraints on `absCommission` when adding a destination chain.

<https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L332>

When adding a new destination chain, the admin will set required parameters like `relCommission` and `absCommission`, which will be used for fee deduction during bridging events. While `relCommission` has a sanity check using `FeeUtils.validateCommission`, `absCommission` is unbounded and can be set to any value.

```solidity
function addDestination(
        bytes32 toChain,
        bytes32 toContract,
        uint16 relCommission,
        uint64 absCommission,
        IAdapter adapter,
        bool requireConsortium
    ) external onlyOwner {
...
        // do not allow 100% commission or higher values
->     FeeUtils.validateCommission(relCommission);

        _getBridgeStorage().destinations[toChain] = DestinationConfig(
            toContract,
            relCommission,
            absCommission,
            adapter,
            requireConsortium
        );
...
    }
```

It is advised to add a sanity check, as this value will be directly added as a fee that users will pay during the deposit event.

```solidiity
function _deposit(
        DestinationConfig memory config,
        bytes32 toChain,
        bytes32 toAddress,
        uint64 amount
    ) internal returns (uint256, bytes memory) {
...
        // relative fee
        uint256 fee = FeeUtils.getRelativeFee(
            amount,
            getDepositRelativeCommission(toChain)
        );
        // absolute fee
->    fee += config.absoluteCommission;
```

## \[N-5] `Height` field from `ValSetAction` struct is never used.

When setting the validator configuration during a call to `Consortium:setInitialValidatorSet`, the data passed will represent a struct:

```solidity
struct ValSetAction {
        uint256 epoch;
        address[] validators;
        uint256[] weights;
        uint256 weightThreshold;
        uint256 height;
    }
```

After validation, the right field will be populated; however, the height field will not be used.

<https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/consortium/Consortium.sol#L91>

```solidity
function setInitialValidatorSet(
        bytes calldata _initialValSet
    ) external onlyOwner {
        // Payload validation
        if (bytes4(_initialValSet) != Actions.NEW_VALSET)
            revert UnexpectedAction(bytes4(_initialValSet));

        ConsortiumStorage storage $ = _getConsortiumStorage();

->    Actions.ValSetAction memory action = Actions.validateValSet(
            _initialValSet[4:]
        );

        if ($.epoch != 0) {
            revert ValSetAlreadySet();
        }

->     _setValidatorSet(
            $,
            action.validators,
            action.weights,
            action.weightThreshold,
            action.epoch
        );
    }
```

If the field is only used for off-chain purposes, it is advised to add a comment explaining its purpose. Otherwise, consider removing the field to simplify the data being passed.

## \[N-6] The invalid `BTCBPMM` documentation does not align with the code and can lead to confusion.

The `BTCBPMM` contract allows users to swap `BTCB` for `LBTC` at a `1:1` rate with a relative fee. To control risk exposure, limits for staking are set and represented by the stakeLimit variable. Documentation present here, along with examples: <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/docs/btcb.md> describes that the staking limit is in `BTCB` tokens. `The BTCBPMM (BTCB Private Market Maker) contract allows users to swap BTCB for LBTC and manages the amount of BTCB that can be staked.`

However, looking at the code, the stake is actually tracked in LBTC minted, rather than BTCB tokens swapped: <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/pmm/BTCB/BTCB.sol#L69>

```solidity
function swapBTCBToLBTC(uint256 amount) external whenNotPaused {
        PMMStorage storage $ = _getPMMStorage();

        ILBTC lbtc = $.lbtc;
        IERC20Metadata btcb = $.btcb;

        uint256 multiplier = $.multiplier;
        uint256 divider = $.divider;
        uint256 amountLBTC = ((amount * multiplier) / divider);
        uint256 amountBTCB = ((amountLBTC * divider) / multiplier);
        if (amountLBTC == 0) revert ZeroAmount();

 ->   if ($.totalStake + amountLBTC > $.stakeLimit)
            revert StakeLimitExceeded();

        // relative fee
        uint256 fee = FeeUtils.getRelativeFee(amountLBTC, $.relativeFee);

->     $.totalStake += amountLBTC;
        btcb.safeTransferFrom(_msgSender(), address(this), amountBTCB);
        lbtc.mint(_msgSender(), amountLBTC - fee);
        lbtc.mint(address(this), fee);
    }
```

This works fine, as those two are represented in a 1:1 ratio. It’s important to note that LBTC has `8` decimals, while BTCB has `18`, which means that during deployment, `stakeLimit` must be set in LBTC, otherwise, this would allow a significantly larger stake amount than intended. Looking at the deployment script file: <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/scripts/deployments/pmm.ts#L21>

```js
task('deploy-pmm', 'Deploys pmm contract via create3')
    .addParam('ledgerNetwork', 'The network name of ledger', 'mainnet')
    .addParam('admin', 'The address of the owner')
    .addParam(
        'proxyFactoryAddr',
        'The ProxyFactory address',
        DEFAULT_PROXY_FACTORY
    )
    .addParam('lbtc', 'The address of the LBTC contract')
    .addParam('btcToken', 'The address of the BTC representation')
-> .addParam('stakeLimit', 'The stake limit', (30n * 10n ** 8n).toString()) // default is 30 LBTC
    .addParam('withdrawAddress', 'The address to withdraw to')
    .addParam('relativeFee', 'The relative fee of the pmm', 10n.toString())
    .addParam('pmm', 'The name of pmm contract')
    .setAction(async (taskArgs, hre) => {
```

there is no issue with parameter settings during deployment as stakeLimit is in LBTC, but it is highly recommended to update the present documentation to avoid further confusion about the intended behaviour.

## \[N-7] Complex bridging flow requires additional documentation.

The Lombard team has invested time in writing good documentation to describe the intended behaviour of the project and provide clear examples of how one can interact with the contracts. However, there are some areas that could benefit from additional details for advanced users, such as the Chainlink CCIP integration with CLAdapter. Although these types of diagrams are never fully complete, the attached image attempts to improve the documentation by providing a contract interaction flow. It is also advised to add additional documentation explaining the off-chain part of the code, including the process of sending transactions and integrating with the Chainlink CCIP system.

## Proof of Concept

## Proof of Concept

These issues are only meant to provide insights for improving the project's quality and documentation.
