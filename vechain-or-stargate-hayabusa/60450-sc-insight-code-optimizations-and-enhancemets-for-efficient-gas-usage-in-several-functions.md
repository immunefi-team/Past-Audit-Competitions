# 60450 sc insight code optimizations and enhancemets for efficient gas usage in several functions

**Submitted on Nov 22nd 2025 at 20:10:35 UTC by @KKam86 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60450
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**

## Description

## Brief/Intro

Following code optimizations may reduce gas costs when making calls to several functions in the contracts:

a) caching storage value outside the loop and then reading it in each iteration

b) correct order of the validations from more basic to more specific

c) using cached output values from one external call rather than making next external calls to the same function

d) unused named return variables which are more gas efficient than no named returns

e) early validations before making writes to the storage

## Vulnerability Details

1. Cache storage value as local variable outside of the loop:

When the length of an storage array or storage value used for iterating is not cached outside of a loop, the Solidity compiler reads these values from storage during each iteration. This results in extra `SLOAD` operations which are more costly than reading from function local variables. Following functions in `Levels` library reads storage value during each iteration in their `for` loops:

```solidity
    function getLevels(
        DataTypes.StargateNFTStorage storage $
    ) external view returns (DataTypes.Level[] memory) {
    >>  DataTypes.Level[] memory levels = new DataTypes.Level[]($.MAX_LEVEL_ID); // SLOAD
    >>  for (uint8 i; i < $.MAX_LEVEL_ID; i++) { // SLOAD in each iteration

    function getLevelsCirculatingSupplies(
        DataTypes.StargateNFTStorage storage $
    ) external view returns (uint208[] memory) {
        uint208[] memory circulatingSupplies = new uint208[]($.MAX_LEVEL_ID);
        for (uint8 i; i < $.MAX_LEVEL_ID; i++) {

    function getLevelsCirculatingSuppliesAtBlock(
        DataTypes.StargateNFTStorage storage $,
        uint48 _blockNumber
    ) external view returns (uint208[] memory) {
        uint208[] memory circulatingSupplies = new uint208[]($.MAX_LEVEL_ID);
        for (uint8 i; i < $.MAX_LEVEL_ID; i++) {

    function _getLevelIds(
        DataTypes.StargateNFTStorage storage $
    ) internal view returns (uint8[] memory) {
        uint8[] memory levelIds = new uint8[]($.MAX_LEVEL_ID);
        for (uint8 i; i < $.MAX_LEVEL_ID; i++) {
```

Consider caching storage `$.MAX_LEVEL_ID` variable before declaring memory array and then use cached value instead reading from storage.

Also caching length of memory array to local variable and then reading this variable is more efficient and bit cheaper than reading the length from memory:

```solidity
    function initializeV3(
        address stargate,
        uint8[] memory levelIds, // memory array
        uint256[] memory boostPricesPerBlock
    ) external onlyRole(UPGRADER_ROLE) reinitializer(3) {
        if (stargate == address(0)) {
            revert Errors.AddressCannotBeZero();
        }

        if (levelIds.length != boostPricesPerBlock.length) {
            revert Errors.ArraysLengthMismatch();
        }

        DataTypes.StargateNFTStorage storage $ = _getStargateNFTStorage();
        $.stargate = IStargate(stargate);
        // @audit length not cached before loop
        for (uint256 i; i < levelIds.length; i++) {
            Levels.updateLevelBoostPricePerBlock($, levelIds[i], boostPricesPerBlock[i]);
        }
    }
```

2. Prefer early validation before reading and writing to storage:

Currently function `addLevel()` in `Levels` library reads and writes from storage before making validation:

```solidity
    function addLevel(
        DataTypes.StargateNFTStorage storage $,
        DataTypes.LevelAndSupply memory _levelAndSupply
    ) external {

>>      $.MAX_LEVEL_ID++; // SSTORE

        // Override level ID to be the new MAX_LEVEL_ID (We do not care about the level id in the input)
>>      _levelAndSupply.level.id = $.MAX_LEVEL_ID; // SLOAD

        // Validate level fields
        _validateLevel(_levelAndSupply.level);

        // Validate supply
        if (_levelAndSupply.circulatingSupply > _levelAndSupply.cap) {
            revert Errors.CirculatingSupplyGreaterThanCap();
        }
```

Consider moving `$.MAX_LEVEL_ID++;` and `_levelAndSupply.level.id = $.MAX_LEVEL_ID;` operations behind validations. Function `_validateLevel()` only validates `name` and `vetAmountRequiredToStake` of the level (`id` is not validated):

```solidity
    function _validateLevel(DataTypes.Level memory _level) internal pure {
        // Name cannot be empty
        if (bytes(_level.name).length == 0) {
            revert Errors.StringCannotBeEmpty();
        }

        // VET amount required to stake must be greater than 0 for all levels except level 0
        if (_level.vetAmountRequiredToStake == 0) {
            revert Errors.ValueCannotBeZero();
        }
    }
```

Early validation is more gas efficient. If function reverts used gas is not returned to the caller. Currently gas is used for incrementing the storage `MAX_LEVEL_ID` value and then reading it when overriding level ID.

3. Incorrect order of validations in `TokenManager.removeTokenManager()` function:

Currently the order of the validation in `removeTokenManager()` is as follows:

```solidity
    function removeTokenManager(DataTypes.StargateNFTStorage storage $, uint256 _tokenId) external {
        address currentManager = $.tokenIdToManager[_tokenId];
        // Check if the caller is the token manager or the token owner
        if (
            currentManager != msg.sender &&
            IStargateNFT(address(this)).ownerOf(_tokenId) != msg.sender
        ) {
            revert Errors.NotTokenManagerOrOwner(_tokenId);
        }
        // Check if the token has no manager
        if (currentManager == address(0)) {
            revert Errors.NoTokenManager(_tokenId);
        }
```

Checking `currentManager` address for not being `address(0)` should be first because:

a) `msg.sender` will never be empty/zero address

b) this will be more gas efficient. Now if function reverts, gas is used first for external call to `ownerOf()` function

4. Unused named return variable in `Stargate.stake()`:

Function `stake()` declares `uint256 tokenId` return variable but not using it when returning:

```solidity
    function stake(
        uint8 _levelId
    ) external payable whenNotPaused nonReentrant returns (uint256 tokenId) { // declared return variable
        ---------
        return $.stargateNFTContract.mint(_levelId, msg.sender); // not using tokenId

```

Consider using declared `uint256 tokenId` variable and change the `return` statement from:

```solidity
return $.stargateNFTContract.mint(_levelId, msg.sender);
```

to:

```solidity
tokenId = $.stargateNFTContract.mint(_levelId, msg.sender);
```

Also according to <https://x.com/DevDacian/status/1796396988659093968> using named return variables is more gas efficient.

5. Use cached balance value instead of another call to `balanceOf()` function:

In function `MintingLogic.boostOnBehalfOf()` output value from `balanceOf()` call is cached to memory value:

```solidity
uint256 balance = $.vthoToken.balanceOf(_sender);
```

Next this cached value is not used in the `if` check and instead next external call to `balanceOf()` is used:

```solidity
        uint256 balance = $.vthoToken.balanceOf(_sender);
        // check that the boost amount is enough
>>>     if ($.vthoToken.balanceOf(_sender) < requiredBoostAmount) {
            revert Errors.InsufficientBalance(
                address($.vthoToken),
                _sender,
                requiredBoostAmount,
                balance
            );
        }
```

Consider using cached balance instead of making another call to `balanceOf()`. If the result of an external call is used multiple times, making the call each time is inefficient and costly.

## Impact Details

Every unnecessary external call or storage write/read operation performed by the function is expensive and makes entire transaction more costly. This should be avoided by applying proper optimisation methods.

## Proof of Concept

## Proof of Concept

1. For example getter function `getLevels()` in `StargateNFT` contract calls external `getLevels()` function from library `Levels`:

```solidity
    function getLevels() external view returns (DataTypes.Level[] memory) {
        return Levels.getLevels(_getStargateNFTStorage());
    }
```

2. `Levels.getLevels()` constructs memory levels array with the use of `for` loop. Number of iterations depends on storage variable `$.MAX_LEVEL_ID`.
3. `$.MAX_LEVEL_ID` variable is type of `uint8` so theoretically the max iterations can be almost `255`. Now each iteration reads $.MAX\_LEVEL\_ID which is `SLOAD` operation in EVM. Minimum gas for this operation according to (for the purpose of this example) <https://www.evm.codes/?fork=prague#54> is `100` while for comparison `MLOAD` minimum gas is only `3` <https://www.evm.codes/?fork=prague#51> .

As we can see reading from memory is much cheaper than reading from storage and reading from local variables is also bit cheaper than reading from memory. In result `for` loop in `getLevels()` should be more efficient when reading `$.MAX_LEVEL_ID` from cached local function variable.
