# #41432 \[SC-High] Attacker can DoS \`StakeV2\`'s rewards distribution by repeatedly inflating Zapper's approval for whitelisted Kodiak Vault tokens

**Submitted on Mar 15th 2025 at 06:23:09 UTC by @merlinboii for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41432
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

An attacker can **repeatedly inflate the Zapper's approval** for any **whitelisted Kodiak Vault tokens** that are **targeted for distribution as `StakeV2` rewards**. This leads to a **Denial-of-Service (DoS) on rewards distribution** due to an **overflow revert in `safeIncreaseAllowance()`**, without incurring any direct cost other than gas fees.

## Vulnerability Details

The **Zapper contract** **increases its allowance** for Kodiak Vault tokens based on **user-specified `amount0Max` and `amount1Max` values**. However, these amounts **are not necessarily fully utilized** when adding liquidity, causing **excess approvals to accumulate indefinitely**.

Over time, this can **inflate the allowance to `uint256.MAX` (or a nearest value), leading to an overflow revert in operation that include calling `safeIncreaseAllowance()`**.\
Note: The attack steps are described in the **Proof of Concept** section.

```solidity
    function _approveAndAddLiquidityToKodiakVault(
        ...
    ) internal returns (uint256, uint256, uint256) {
        --- SNIPPED ---
@>      token0.safeIncreaseAllowance(address(kodiakStakingRouter), stakingParams.amount0Max);
@>      token1.safeIncreaseAllowance(address(kodiakStakingRouter), stakingParams.amount1Max);
        // add liquidity using KodiakStakingRouter
        return kodiakStakingRouter.addLiquidity(
            IKodiakVaultV1(kodiakVault),
            stakingParams.amount0Max,
            stakingParams.amount1Max,
            stakingParams.amount0Min,
            stakingParams.amount1Min,
            stakingParams.amountSharesMin,
            stakingParams.receiver
        );
    }
```

```solidity
// SafeERC20.sol
function safeIncreaseAllowance(IERC20 token, address spender, uint256 value) internal {
    uint256 oldAllowance = token.allowance(address(this), spender);
@>  forceApprove(token, spender, oldAllowance + value);
}
```

The actual amount of tokens used is computed within `IslandRouter._addLiquidity()`, where `amount0In` and `amount1In` are determined based on the minimum amount from `_computeMintAmounts()`.

```solidity
// KodiakIslandRouter: 0x679a7C63FC83b6A4D9C1F931891d705483d4791F (Berachain-Mainnet)
function _addLiquidity(
    ...
) internal returns (uint256 amount0, uint256 amount1, uint256 mintAmount) {
    IERC20 token0 = island.token0();
    IERC20 token1 = island.token1();
@>  (uint256 amount0In, uint256 amount1In, uint256 _mintAmount) = island.getMintAmounts(amount0Max, amount1Max);
    require(amount0In >= amount0Min && amount1In >= amount1Min && _mintAmount >= amountSharesMin, "below min amounts");

@>  if (amount0In > 0) token0.safeTransferFrom(msg.sender, address(this), amount0In);
@>  if (amount1In > 0) token1.safeTransferFrom(msg.sender, address(this), amount1In);

    return _deposit(island, amount0In, amount1In, _mintAmount, receiver);
}
```

```solidity
// YEET-WBERA KodiakIsland: 0xEc8BA456b4e009408d0776cdE8B91f8717D13Fa1 (Berachain-Mainnet)

function getMintAmounts(uint256 amount0Max, uint256 amount1Max) external view returns (uint256 amount0, uint256 amount1, uint256 mintAmount) {
    uint256 totalSupply = totalSupply();
    if (totalSupply > 0) {
@>      (amount0, amount1, mintAmount) = _computeMintAmounts(totalSupply, amount0Max, amount1Max);
    } else { //@note assume that it on-live and totalSupply > 0
        // ...
    }
}

function _computeMintAmounts(uint256 totalSupply, uint256 amount0Max, uint256 amount1Max) private view returns (uint256 amount0, uint256 amount1, uint256 mintAmount) {
    (uint256 amount0Current, uint256 amount1Current) = getUnderlyingBalances();

    --- SNIPPED ---
    } else {
        // only if both are non-zero
        uint256 amount0Mint = FullMath.mulDiv(amount0Max, totalSupply, amount0Current);
        uint256 amount1Mint = FullMath.mulDiv(amount1Max, totalSupply, amount1Current);
        require(amount0Mint > 0 && amount1Mint > 0, "mint 0");

@>      mintAmount = amount0Mint < amount1Mint ? amount0Mint : amount1Mint;
    }

    // compute amounts owed to contract
@>  amount0 = FullMath.mulDivRoundingUp(mintAmount, amount0Current, totalSupply);
@>  amount1 = FullMath.mulDivRoundingUp(mintAmount, amount1Current, totalSupply);
}
```

## Impact Details

* The `StakeV2` reward distribution suffers from a **Denial-of-Service (DoS) vulnerability**, preventing rewards from being distributed, which results in the **permanent freezing of unclaimed yield**.
* The **Zapper** cannot execute `_yeetIn` for certain tokens that have undergone **inflation in approval amounts**.

Although the **Kodiak Router** can be replaced with a new one, the attack can be **repeated** on the new router, as the logic still allows it.

Moreover, in the rare case that the **Kodiak Vault** settings in the **Zapper contract** become malicious, an attacker could potentially **pull any funds from the Zapper** (if funds are available).

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L507-L508\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L153-L180\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L182-L201>

## Link to Proof of Concept

<https://gist.github.com/merlinboii/90dfcadc9170a55a3e6b39555a4e0461>

## Proof of Concept

## Proof of Concept

* Here is the **runnable PoC**: <https://gist.github.com/merlinboii/90dfcadc9170a55a3e6b39555a4e0461\\>
  Result Logs:

```bash
[PASS] test_attack_inflateApproval_DoS() (gas: 5141513)
Logs:
  Inflated allowance: 115792089237316195423570985008687907853269984665640564039457534956561230671075
  Delta: 49051351898968860
```

* Below is the step-by-step conceptual PoC: This PoC uses YEET-WBERA KodiakIsland ([0xEc8BA456b4e009408d0776cdE8B91f8717D13Fa1](https://beratrail.io/address/0xEc8BA456b4e009408d0776cdE8B91f8717D13Fa1/contract/80094/code)) on Berachain Mainnet to demonstrate how a user can inflate Zapper’s approval allowance without spending full amounts.

1. **`StakeV2` Rewards System**
   * The **LP rewards** for **YEET-WBERA Kodiak Island** consist of:
     * `token0`: **YEET**
     * `token1`: **WBERA**
   * The `StakeV2` contract contains the functions:
     * `executeRewardDistributionYeet()`, which distributes **YEET** rewards.
     * `executeRewardDistribution()`, which distributes **BERA** rewards.
   * These functions call `_yeetIn()` to **add liquidity** to the **YEET-WBERA Kodiak Vault** and **deposit LP rewards into Trifecta Vaults**.
2. **Attacker Target: DoS the Reward Distribution and Feasibility**

   * **Approach for DoS**: Inflate the Zapper's approval allowance for `YEET` or `WBERA` to `uint256(MAX)`, causing an overflow revert when `safeIncreaseAllowance()` is executed before liquidity is added.

     ```solidity
     function _approveAndAddLiquidityToKodiakVault(
         ...
     ) internal returns (uint256, uint256, uint256) {
         --- SNIPPED ---
     ```

   @> token0.safeIncreaseAllowance(address(kodiakStakingRouter), stakingParams.amount0Max);\
   @> token1.safeIncreaseAllowance(address(kodiakStakingRouter), stakingParams.amount1Max);\
   // add liquidity using KodiakStakingRouter\
   return kodiakStakingRouter.addLiquidity(\
   IKodiakVaultV1(kodiakVault),\
   stakingParams.amount0Max,\
   stakingParams.amount1Max,\
   stakingParams.amount0Min,\
   stakingParams.amount1Min,\
   stakingParams.amountSharesMin,\
   stakingParams.receiver\
   );\
   }\
   \`\`\`

   * **Approach for Attack Feasibility**: Target Zapper functions that **do not require transferring full `stakingParams.amount0Max` or `stakingParams.amount1Max` before executing `_yeetIn()`**:
     * `zapInToken0()`: Transfers `stakingParams.amount0Max + swapData.inputAmount` and allows **arbitrary input for `stakingParams.amount1Max`**.
     * `zapInToken1()`: Transfers `stakingParams.amount1Max + swapData.inputAmount` and allows **arbitrary input for `stakingParams.amount0Max`**.
     * `zapIn()`: Transfers the `inputToken` amount and allows **arbitrary input for `stakingParams.amount0Max` and `stakingParams.amount1Max`**.
     * Since the `addLiquidity()` process caps minting at the **minimum liquidity result** calculated from both `amount0Max` and `amount1Max`, the **arbitrary inputs will not affect the actual LP minting results**.
3. Attacker calls `Zapper.zapInToken0()`. The user **sets an artificially high approval of `stakingParams.amount1Max`** for Kodiak Vault tokens (`YEET` and `WBERA`):

```solidity
Zapper.zapInToken0(
    swapData, // swap inputAmount: YEET (token0) for WBERA (token1), preparing WBERA for adding liquidity along with YEET from `stakingParams.amount0Max`
    KodiakVaultStakingParams({
        kodiakVault: address(YEET_BERA_KODIAK_ISLAND),
        amount0Max: 1e18, // 1 YEET
        amount1Max: 7719472615821079694904732333912527190217998977709370935963838933860875309329, // uint256(MAX) / 15 times 
        // ...
        receiver: address(user) // user got LP directly and not trigger to deposit them to vault
    }),
    vaultParams
)
```

3. The process triggers to **increase allowance** up to
   * `amount0Max: 1e18`
   * `amount1Max: 7719472615821079694904732333912527190217998977709370935963838933860875309329`.

```solidity
    function _approveAndAddLiquidityToKodiakVault(
        address kodiakVault,
        IERC20 token0,
        IERC20 token1,
        IZapper.KodiakVaultStakingParams calldata stakingParams
    ) internal returns (uint256, uint256, uint256) {
        --- SNIPPED ---
@>      token0.safeIncreaseAllowance(address(kodiakStakingRouter), stakingParams.amount0Max);
@>      token1.safeIncreaseAllowance(address(kodiakStakingRouter), stakingParams.amount1Max);
        // add liquidity using KodiakStakingRouter
        return kodiakStakingRouter.addLiquidity(
            IKodiakVaultV1(kodiakVault),
            stakingParams.amount0Max,
            stakingParams.amount1Max,
            stakingParams.amount0Min,
            stakingParams.amount1Min,
            stakingParams.amountSharesMin,
            stakingParams.receiver
        );
    }
```

3. We can see the actual of `amount0In`, `amount1In` and `mintAmount` from that simulate block (`Block: 2275117`) by reading from `YEET-WBERA-KodiakIsland.getMintAmounts()`:

```
[
  "999999999999999986", //amount0In
  "3461048089839568", //amount1In
  "43710258853688038" //mintAmount
]
```

3. As from the process of `IslandRouter._addLiquidity()`, it will only transfer `999999999999999986` YEET and `3461048089839568` WBERA to the router contract for further deposit operation (minting LP).

```solidity
function _addLiquidity(
    ...
) internal returns (uint256 amount0, uint256 amount1, uint256 mintAmount) {
    IERC20 token0 = island.token0();
    IERC20 token1 = island.token1();
    (uint256 amount0In, uint256 amount1In, uint256 _mintAmount) = island.getMintAmounts(amount0Max, amount1Max);
    require(amount0In >= amount0Min && amount1In >= amount1Min && _mintAmount >= amountSharesMin, "below min amounts");

@>  if (amount0In > 0) token0.safeTransferFrom(msg.sender, address(this), amount0In);
@>  if (amount1In > 0) token1.safeTransferFrom(msg.sender, address(this), amount1In);

    return _deposit(island, amount0In, amount1In, _mintAmount, receiver);
}
```

4. **The allowance of `Zapper` for `KodiakStakingRouter` will not reset to 0 and remains**:

* YEET: 1e18 - 999999999999999986
* WBERA: 7719472615821079694904732333912527190217998977709370935963838933860875309329 - 3461048089839568 = **\~uint256(MAX) / 15 times**

5. The attacker can **repeat the process** until the allowance reaches `uint256.MAX` (or the nearest value), causing an **overflow revert** whenever the target token executes `safeIncreaseAllowance()` on the Zapper for a Kodiak router.
6. The attacker **receives LP tokens** along with unused `YEET` and `WBERA` from providing liquidity. **This means the attacker only incurs gas costs for execution.**
7. When `StakeV2.executeRewardDistributionYeet()` or `StakeV2.executeRewardDistribution()` is executed, and the `oldAllowance` (inflated to \~`uint256.MAX`) plus `value` (`stakingParams.amountXMax`) **exceeds `uint256.MAX`**, it **triggers an overflow revert**:

```solidity
    function safeIncreaseAllowance(IERC20 token, address spender, uint256 value) internal {
        uint256 oldAllowance = token.allowance(address(this), spender);
@>      forceApprove(token, spender, oldAllowance + value);
    }
```

### Final Stage: Denial-of-Service (DoS) to StakeV2 Reward Distribution process via Zapper Approval Inflation

The attacker **inflates the Zapper’s approval for `YEET` and `WBERA` on the `KodiakStakingRouter`**, **causing a DoS condition by making further approvals impossible**.
