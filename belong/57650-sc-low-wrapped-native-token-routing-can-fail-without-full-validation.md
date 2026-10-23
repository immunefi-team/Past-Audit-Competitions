# 57650 sc low wrapped native token routing can fail without full validation

**Submitted on Oct 27th 2025 at 20:56:34 UTC by @Oxv1bh4 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57650
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The `_buildPath` function constructs a Uniswap V3 swap path dynamically, choosing either a direct pool or routing through a configured wrapped native token. However, in the two-hop scenario, the function only checks if the first pool exists (`tokenIn → W_NATIVE`) and assumes the second pool (`W_NATIVE → tokenOut`) exists. If the second pool does not exist, the swap will revert at execution, causing the transaction to fail.

### Vulnerability Details

In the current implementation, the function only validates the existence of the first hop in a two-hop swap path and assumes that the second hop (from the wrapped native token to the final output token) exists. As a result, the function can produce a path that is invalid, leading to failed swaps at runtime when executed through the Uniswap V3 router. If the second pool does not exist, a swap call using this path will revert, blocking users from completing the transaction.

Code excerpt:

```solidity
function _buildPath(PaymentsInfo memory _paymentsInfo, address tokenIn, address tokenOut)
        internal
        view
        returns (bytes memory path)
    {
        // Direct pool
        if (
            IV3Factory(_paymentsInfo.swapV3Factory).getPool(tokenIn, tokenOut, _paymentsInfo.swapPoolFees) != address(0)
        ) {
            path = abi.encodePacked(tokenIn, _paymentsInfo.swapPoolFees, tokenOut);
        }
        // tokenIn -> W_NATIVE_CURRENCY -> tokenOut
        // @audit - W_NATIVE_CURRENCY to tokenOut pool validation is missing
        else if (
            IV3Factory(_paymentsInfo.swapV3Factory)
                    .getPool(tokenIn, _paymentsInfo.wNativeCurrency, _paymentsInfo.swapPoolFees) != address(0)
        ) {
            path = abi.encodePacked(
                tokenIn, _paymentsInfo.swapPoolFees, _paymentsInfo.wNativeCurrency, _paymentsInfo.swapPoolFees, tokenOut
            );
        } else {
            revert NoValidSwapPath();
        }
    }
```

## Impact Details

Failed swaps stop the protocol from sending the expected output tokens to promoters, escrows, or venues, resulting in the protocol not delivering the promised returns. The issue is classified as LOW severity because it prevents the expected transfer but does not result in direct loss of funds from the protocol.

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol?utm\\_source=immunefi#L741-L763>

## Proof of Concept

The following steps reproduce the issue by mocking Uniswap V3 components and showing a missing second hop causes a swap to fail at execution time.

{% stepper %}
{% step %}

### 1) Add mock Uniswap V3 contracts

Create `contracts/mocks/MockUniswapV3.sol` with:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/*//////////////////////////////////////////////////////////////
                         MOCK UNISWAP V3 FACTORY
//////////////////////////////////////////////////////////////*/

contract MockUniswapV3Factory {
    mapping(bytes32 => address) public pools;

    /// @notice Registers a fake pool between two tokens
    function setPool(address tokenA, address tokenB, uint24 fee) external {
        bytes32 key = _pairKey(tokenA, tokenB, fee);
        // Create a deterministic fake pool address
        address fakePool = address(uint160(uint256(keccak256(abi.encodePacked(tokenA, tokenB, fee)))));
        pools[key] = fakePool;
    }

    /// @notice Returns pool address for given token pair and fee
    function getPool(address tokenA, address tokenB, uint24 fee) external view returns (address) {
        return pools[_pairKey(tokenA, tokenB, fee)];
    }

    function _pairKey(address tokenA, address tokenB, uint24 fee) internal pure returns (bytes32) {
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        return keccak256(abi.encodePacked(token0, token1, fee));
    }
}

/*//////////////////////////////////////////////////////////////
                         MOCK UNISWAP V3 QUOTER
//////////////////////////////////////////////////////////////*/

contract MockUniswapV3Quoter {
    /// @notice Returns a constant quote equal to the input
    function quoteExactInput(bytes calldata, uint256 amountIn) external pure returns (uint256) {
        return amountIn;
    }

    /// @notice Mimics slippage adjustment for convenience
    function amountOutMin(uint256 slippageBps) external pure returns (uint256) {
        // Just return a dummy value slightly less than input
        return (10000 - slippageBps);
    }
}

/*//////////////////////////////////////////////////////////////
                         MOCK UNISWAP V3 ROUTER
//////////////////////////////////////////////////////////////*/

interface IFactoryLike {
    function getPool(address tokenA, address tokenB, uint24 fee) external view returns (address);
}

contract MockUniswapV3Router {
    address public factory;

    event SwapExecuted(address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOut);

    constructor(address _factory) {
        factory = _factory;
    }

    /// @notice Simulates a swap along the path, reverting if any pool is missing
    function exactInput(bytes calldata path, address recipient, uint256 amountIn, uint256 amountOutMinimum)
        external
        returns (uint256 amountOut)
    {
        // Decode and verify every hop in the path
        (address[] memory tokens, uint24[] memory fees) = _decodePath(path);

        for (uint256 i; i < tokens.length - 1; ++i) {
            address pool = IFactoryLike(factory).getPool(tokens[i], tokens[i + 1], fees[i]);
            if (pool == address(0)) revert MissingPool(tokens[i], tokens[i + 1]);
        }

        // Simulate a successful swap if all pools exist
        amountOut = amountIn - (amountIn * 5 / 1000); // 0.5% "fee"
        emit SwapExecuted(tokens[0], tokens[tokens.length - 1], amountIn, amountOut);

        // Just return the simulated amountOut
        require(amountOut >= amountOutMinimum, "SlippageTooHigh");
        return amountOut;
    }

    /*//////////////////////////////////////////////////////////////
                            INTERNAL HELPERS
    //////////////////////////////////////////////////////////////*/

    function _decodePath(bytes memory path) internal pure returns (address[] memory tokens, uint24[] memory fees) {
        // Each hop = token(20) + fee(3)
        uint256 numPools = (path.length - 20) / 23;
        tokens = new address[](numPools + 1);
        fees = new uint24[](numPools);

        uint256 offset = 0;
        for (uint256 i; i < numPools; ++i) {
            address tokenA;
            uint24 fee;
            address tokenB;

            assembly {
                tokenA := mload(add(path, add(20, offset)))
                fee := mload(add(path, add(23, offset)))
                tokenB := mload(add(path, add(43, offset)))
            }

            tokens[i] = tokenA;
            fees[i] = fee;
            if (i == numPools - 1) tokens[numPools] = tokenB;

            offset += 23;
        }
    }

    error MissingPool(address tokenA, address tokenB);
}
```

{% endstep %}

{% step %}

### 2) Add helper to set up mocks in tests

Add the following helper to `test/v2/platform/belong-check-in.test.ts`:

```ts
async function setUpMockUniswap() {  
    // Deploy factory
    const MockV3Factory = await ethers.getContractFactory("MockUniswapV3Factory");
    const mockFactory = await MockV3Factory.deploy();
    await mockFactory.deployed();
  
    // Deploy quoter
    const MockV3Quoter = await ethers.getContractFactory("MockUniswapV3Quoter");
    const mockQuoter = await MockV3Quoter.deploy();
    await mockQuoter.deployed();
  
    // Deploy router linked to factory
    const MockV3Router = await ethers.getContractFactory("MockUniswapV3Router");
    const mockRouter = await MockV3Router.deploy(mockFactory.address);
    await mockRouter.deployed();
  
    // Register only one pool (USDC → WETH), simulate missing second hop
    await mockFactory.setPool(USDC_ADDRESS, WETH_ADDRESS, POOL_FEE);
  
    return {
      v3Factory: mockFactory.address,
      v3Quoter: mockQuoter.address,
      v3Router: mockRouter.address,
    };
  }
```

{% endstep %}

{% step %}

### 3) Update the test fixture to inject mock Uniswap addresses

Update `fixture` in `test/v2/platform/belong-check-in.test.ts` to call `setUpMockUniswap()` and set `mockPaymentsInfo` using returned addresses:

```ts
async function fixture() {
    const [admin, treasury, manager, minter, burner, pauser, referral] = await ethers.getSigners();
    const signer = EthCrypto.createIdentity();

    const WETH_whale = await getSignerFromAddress(WETH_WHALE_ADDRESS);
    const USDC_whale = await getSignerFromAddress(USDC_WHALE_ADDRESS);
    const ENA_whale = await getSignerFromAddress(ENA_WHALE_ADDRESS);
    const WETH = await getToken(WETH_ADDRESS);
    const USDC = await getToken(USDC_ADDRESS);
    const ENA = await getToken(ENA_ADDRESS);

    const signatureVerifier: SignatureVerifier = await deploySignatureVerifier();
    const validator: MockTransferValidatorV2 = await deployMockTransferValidatorV2();
    const accessTokenImplementation: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
    const royaltiesReceiverV2Implementation: RoyaltiesReceiverV2 = await deployRoyaltiesReceiverV2Implementation();
    const creditTokenImplementation: CreditToken = await deployCreditTokenImplementation();
    const vestingWallet: VestingWalletExtended = await deployVestingWalletImplementation();

    const treasuryUsdcBalance = await USDC.balanceOf(treasury.address);
    if (!treasuryUsdcBalance.isZero()) {
      await USDC.connect(treasury).transfer(USDC_whale.address, treasuryUsdcBalance);
    }

    implementations = {
      accessToken: accessTokenImplementation.address,
      creditToken: creditTokenImplementation.address,
      royaltiesReceiver: royaltiesReceiverV2Implementation.address,
      vestingWallet: vestingWallet.address,
    };

    const factory: Factory = await deployFactory(
      treasury.address,
      signer.address,
      signatureVerifier.address,
      validator.address,
      implementations,
    );

    const helper: Helper = await deployHelper();
    const staking: Staking = await deployStaking(admin.address, treasury.address, ENA_ADDRESS);

    const referralCode = EthCrypto.hash.keccak256([
      { type: 'address', value: referral.address },
      { type: 'address', value: factory.address },
      { type: 'uint256', value: chainId },
    ]);

    await factory.connect(referral).createReferralCode();

    const { v3Factory, v3Quoter, v3Router } = await setUpMockUniswap();

    const mockPaymentsInfo: BelongCheckIn.PaymentsInfoStruct = {
      swapPoolFees: POOL_FEE,
      slippageBps: BigNumber.from(10).pow(27).sub(1),
      swapV3Factory: v3Factory,
      swapV3Router: v3Router,
      swapV3Quoter: v3Quoter,
      wNativeCurrency: WETH_ADDRESS,
      usdc: USDC_ADDRESS,
      long: ENA_ADDRESS,
      maxPriceFeedDelay: MAX_PRICEFEED_DELAY,
    };

    const belongCheckIn: BelongCheckIn = await deployBelongCheckIn(
      signatureVerifier.address,
      helper.address,
      admin.address,
      mockPaymentsInfo,
    );

    const escrow: Escrow = await deployEscrow(belongCheckIn.address);
    const { pf1, pf2, pf2_2, pf2_3, pf3 } = await deployPriceFeeds();
    const { venueToken, promoterToken } = await deployCreditTokens(
      true,
      false,
      factory.address,
      signer.privateKey,
      admin,
      manager.address,
      belongCheckIn.address,
      belongCheckIn.address,
    );

    contracts = {
      factory: factory.address,
      escrow: escrow.address,
      staking: staking.address,
      venueToken: venueToken.address,
      promoterToken: promoterToken.address,
      longPF: pf1.address,
    };

    await belongCheckIn.setContracts(contracts);

    return {
      signatureVerifier,
      helper,
      factory,
      staking,
      venueToken,
      promoterToken,
      belongCheckIn,
      escrow,
      pf1,
      pf2,
      pf2_2,
      pf2_3,
      pf3,
      admin,
      treasury,
      manager,
      minter,
      burner,
      pauser,
      referral,
      signer,
      referralCode,
      WETH,
      USDC,
      ENA,
      WETH_whale,
      USDC_whale,
      ENA_whale,
    };
  }
```

{% endstep %}

{% step %}

### 4) Add the test case that demonstrates the failing swap

Add this test inside the `Venue Flow` describe block in `test/v2/platform/belong-check-in.test.ts`:

```ts
it.only('Wrapped Native Token Routing Can Fail Without Full Validation', async () => {
      // Load fixture with deployed contracts and test accounts
      const { belongCheckIn, signer, USDC, USDC_whale } = await loadFixture(fixture);
    
      // Define parameters for venue deposit
      const uri = 'uriuri';
      const amount = await u(100, USDC);
      const venue = USDC_whale.address;
    
      // Generate signature authorizing the venue deposit
      const message = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const signature = EthCrypto.sign(signer.privateKey, message);
    
      // Venue info struct for the deposit call
      const venueInfo: VenueInfoStruct = {
        rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,
        venue,
        amount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature,
      };
    
      // Approve total amount for the deposit including convenience fee
      const willBeTaken = convenienceFeeAmount.add(amount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
    
      // Should not revert with `NoValidSwapPath` since intermediate pool validation is missing
      await expect(
        belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo)
      ).to.not.be.revertedWithCustomError(belongCheckIn, 'NoValidSwapPath');
    
      // Should revert with `SwapFailed` due to missing intermediate pool between W_NATIVE and tokenOut
      await expect(
        belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo)
      ).to.be.revertedWithCustomError(belongCheckIn, 'SwapFailed');
    });
```

{% endstep %}

{% step %}

### 5) Run tests

Run:

```
npm run test
```

Expected behavior demonstrated by the test:

* The path builder does not revert with `NoValidSwapPath` because it validated only the first hop.
* The Uniswap router simulation reverts during swap execution due to the missing second hop, causing `SwapFailed`.
  {% endstep %}
  {% endstepper %}

Notes:

* The root cause is missing validation of the second hop (W\_NATIVE → tokenOut) when building a two-hop path.
* Fixing the issue requires adding an extra pool existence check for the second hop before returning the two-hop encoded path. The report does not include a suggested patch; it only documents the detection and reproduction.
