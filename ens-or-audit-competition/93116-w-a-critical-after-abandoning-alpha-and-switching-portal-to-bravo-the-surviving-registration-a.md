# 93116 w a critical after abandoning alpha and switching portal to bravo the surviving registration actor later submits register alpha while portal displays bravo

**Submitted on Sep 14th 2026 at 09:56:43 UTC by @manvi for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93116
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief / Intro

I analysed the Portal registration lifecycle and found a deterministic cross-name actor-lifecycle bug that violates the **registration-name / reviewed-resource integrity boundary**.

Portal's `/register` page stores the selected name in a **search parameter**. `RegisterName` creates `useRegistrationTransactions({ name, duration })`, and that hook creates one XState `registrationMachine` actor with:

```tsx
useActorRef(registrationMachine, {
  input: { chainId },
})
```

The actor is **not keyed by name**.

When registration starts for Alpha, the machine snapshots:

```
context.name = Alpha
```

and independently progresses through resolver deployment, commitment, approval, commitment cooldown, and final registration.

While the machine is in `commitmentCooldown`, the user can close the transaction modal and use Portal's sidebar search to navigate from:

```
/register?name=Alpha
```

to:

```
/register?name=Bravo
```

Portal explicitly asks whether the user wants to leave the in-progress registration. If the user accepts, Portal updates the current registration page and checkout to Bravo.

However:

* the navigation path does **not** send `CANCEL` to the Alpha registration actor;
* the app-wide route-reset hook only closes the transaction modal;
* closing the transaction modal does not cancel pending registration state;
* the registration actor still contains `context.name = Alpha`;
* after the commitment cooldown expires, the machine autonomously enters `registeringDomain`;
* `submitRegistrationActor()` encodes `context.name` into the actual registrar call;
* `transactionManager.startTransaction()` immediately starts the prebuilt request;
* the connected wallet is therefore asked to submit `register(Alpha, ...)` while Portal is displaying Bravo.

The browser-to-chain PoC reproduced the wrong-name transaction and its economic consequence:

```
[AUDIT SWITCH]
current=portal-bravo-mtwxqxch.eth
Bravo-total-usd=8.01
abandoned-Alpha-total-usd=160.11

[AUDIT STALE REQUEST]
current-ui=portal-bravo-mtwxqxch.eth
current-total-usd=8.01
pending-register-label=pxch
wallet-send-pending=1

[AUDIT POST]
Alpha=pxch.eth available=false
Bravo=portal-bravo-mtwxqxch.eth available=true
current-Bravo-total-usd=8.01
actual-USDC-debit=160.109598
```

The E2E completed with:

```
1 passed (1.3m)
POC RETURN CODE: 0
```

This Portal reproduction does **not** depend on delaying or reordering an RPC response. Once Alpha has reached its ordinary commitment cooldown, the stale actor survives normal Portal navigation and later wakes on its own.

***

## Vulnerability Details

### 1. Portal represents the registration name as a search parameter on one stable `/register/` route

Affected file:

```
apps/portal/src/routes/register/index.tsx
```

Frozen source:

```tsx
15 const RegisterPage = () => {
16   const { name } = Route.useSearch()
...
39   ) : (
40     <RegisterName name={name} />
41   )}
...
48 export const Route = createFileRoute('/register/')({
49   component: RegisterPage,
50   validateSearch: (search: Record<string, unknown>): RegisterSearch => {
51     const rawName = typeof search.name === 'string' ? search.name : undefined
52     const name = rawName?.trim() && rawName !== '.eth' ? rawName : undefined
53     return {
54       name,
55     }
56   },
57 })
```

The resource identity is therefore:

```
route = /register/
name  = search parameter
```

A name change does not inherently imply a different route component.

### 2. Portal's own search performs an in-app `/register` search-parameter transition

Affected file:

```
apps/portal/src/features/dashboard/components/HomeSearchInput.tsx
```

Frozen source:

```tsx
76 const navigateToRegister = useCallback(
77   (name: string) => {
78     const nameWithEth = name.includes('.') ? name : `${name}.eth`
79     navigate({ to: '/register', search: { name: nameWithEth } })
80   },
81   [navigate],
82 )
```

This is the exact Alpha → Bravo navigation used by the PoC. It is not a full page reload.

### 3. `useRegistrationTransactions` creates the registration actor without binding its lifecycle to `name`

Affected file:

```
apps/portal/src/features/register/hooks/useRegistrationTransactions.ts
```

Frozen source:

```tsx
66 export const useRegistrationTransactions = ({
67   name,
68   duration,
69 }: UseRegistrationTransactionsParams) => {
...
80   const actor: RegistrationMachineActor = useActorRef(registrationMachine, {
81     input: { chainId },
82   })
```

The actor input is:

```
chainId
```

not:

```
chainId + current registration name
```

There is also no effect in this hook that watches `name` and cancels/recreates the actor when the selected name changes.

This is the lifecycle root of the bug.

### 4. Alpha is snapshotted into authoritative registration-machine context only when the flow starts

The Portal hook sends:

```tsx
167 actor.send({
168   type: 'START_REGISTRATION',
169   name,
170   duration: BigInt(duration),
171   token: savedParams.tokenSymbol,
172   price: savedParams.tokenPrice,
173   signer,
174   accountAddress: connection.address,
175   publicClient,
176 })
```

The shared registration machine permanently snapshots that resource:

```
packages/transaction-manager/src/machines/registration/registration.machine.ts
```

```ts
539 START_REGISTRATION: {
540   target: 'settingUpRegistration',
541   actions: assign({
542     name: ({ event }) => event.name,
543     duration: ({ event }) => event.duration,
...
```

For an Alpha registration:

```
context.name = Alpha
```

Changing the React `name` prop later does not mutate this context.

### 5. The commitment itself is built from the snapshotted Alpha context

The machine's commitment preparation consumes:

```ts
910 preparingCommitment: {
...
921   return {
922     name: context.name,
923     owner: context.ownerAddress ?? context.accountAddress!,
924     duration: context.duration,
...
```

The same `context.name` is therefore used to create Alpha's real commitment.

This matters because the later final Alpha registration is not malformed. The old actor has a valid Alpha commitment and can successfully reveal/register Alpha after the cooldown.

### 6. Portal explicitly allows the user to leave the in-progress Alpha flow

Affected file:

```
apps/portal/src/features/register/components/RegisterName.tsx
```

Frozen source:

```tsx
67 // Prevent navigation during registration
68 useBlocker({
69   shouldBlockFn: () => {
70     if (!isRegistering) return false
71     const shouldLeave = confirm(
72       'Your registration is in progress. Leaving may interrupt it and you could lose your commitment. Are you sure you want to leave?',
73     )
74     return !shouldLeave
75   },
76 })
```

The warning establishes the intended user-facing meaning of accepting navigation:

```
leaving may INTERRUPT registration
and may LOSE the commitment
```

It does not say:

```
the abandoned registration will continue in the background
and may later submit its final purchase while another name is current
```

The successful PoC captured exactly this warning and accepted it:

```
[AUDIT BLOCKER] Your registration is in progress. Leaving may interrupt it and you could lose your commitment. Are you sure you want to leave?
```

### 7. Accepting the navigation warning does not cancel Alpha

The only explicit reset helper in the registration hook is:

```tsx
315 const resetRegistration = useCallback(() => {
316   actor.send({ type: 'CANCEL' })
317   closeModal()
318   clearTransaction()
319 }, [actor, closeModal, clearTransaction])
```

But normal Alpha → Bravo navigation does not call this helper.

The source search shows `resetRegistration` is created/exported by the hook but is not used by the Portal name-navigation path.

So:

```
user accepts "leave registration"
!=
registration actor CANCEL
```

### 8. Portal's global route-reset logic closes only the modal, not the registration actor

Affected file:

```
apps/portal/src/features/transaction-manager/hooks/useTransactionModal.ts
```

Frozen source:

```tsx
26 /**
27  * Closes the transaction modal whenever the route changes.
...
36 export const useTransactionModalRouteReset = () => {
37   const router = useRouter()

39   useEffect(
40     () => router.subscribe('onResolved', () => closeTransactionModal()),
41     [router],
42   )
43 }
```

This handler only changes the app-wide modal atom.

It does **not**:

* send `CANCEL` to the registration machine;
* clear the machine's `context.name`;
* clear the Alpha commitment;
* invalidate the actor;
* bind the actor to the new search parameter.

### 9. Closing the transaction modal also intentionally preserves non-failed transaction state

Affected file:

```
apps/portal/src/features/transaction-manager/components/TransactionModal.tsx
```

Frozen source:

```tsx
47 const handleClose = () => {
48   closeModal()
49 }
...
55 open={isOpen}
56 onOpenChange={(open) => {
57   if (!open) {
58     handleClose()
59     // Only clear when transaction has failed - don't clear pending (tx may still be in wallet)
60     // or success (user might want to reopen and see the result)
61     if (txState?.error) {
62       clearTransaction()
63     }
64   }
65 }}
```

The modal close operation is therefore presentation state, not registration cancellation.

The PoC closes the modal after Alpha reaches `commitmentCooldown`; the Alpha actor remains alive.

### 10. After navigation, Portal's visible registration form and checkout are rebuilt from current Bravo

`RegisterName` consumes the current route/search `name` prop:

```tsx
153 return (
154   <main ...>
...
161     <RegisterNameForm
162       name={name}
163       duration={duration}
...
167     <RegisterNameCheckoutSummary name={name} duration={duration} />
168     <PaymentTokenSection
...
173       name={name}
174       duration={duration}
...
```

The checkout summary directly prices the current prop:

```
apps/portal/src/features/register/components/RegisterNameSummary.tsx
```

```tsx
47 export const RegisterNameCheckoutSummary = ({
48   name,
49   duration,
50 }: RegisterNameCheckoutSummaryProps) => {
...
58   } = useQuery({
59     ...getRegistrationPriceQueryOptions({
60       name,
61       duration,
62     }),
...
```

and renders:

```tsx
279 <SummaryRow
280   label="Total"
281   value={formatRegistrationTotal(
282     price.base,
283     price.premium,
284     price.decimals,
285   )}
...
```

Therefore, after Alpha → Bravo:

```
current route/search name = Bravo
current visible form name = Bravo
current checkout price    = Bravo
old registration actor    = Alpha
```

The successful run measured:

```
Alpha displayed total = $160.11
Bravo displayed total = $8.01
```

### 11. Portal's modal transaction descriptors also rebuild from the current Bravo prop

Within `useRegistrationTransactions`, the modal transaction descriptions and preview intent use the live React `name`:

```tsx
192 const transactions: Transaction[] = useMemo(() => {
...
195     id: REGISTRATION_TX_IDS.deployResolver,
196     title: 'Deploy resolver',
197     transactionName: `Deploy resolver for ${name}`,
...
222     id: REGISTRATION_TX_IDS.commit,
223     title: 'Submit commitment',
224     transactionName: `Commit to register ${name}`,
...
256 steps.push({
257   id: REGISTRATION_TX_IDS.register,
258   title: 'Register name',
259   transactionName: `Register ${name}`,
...
268       ...encodeRegisterCall({
269         name,
...
```

After the search parameter becomes Bravo, these UI/preview descriptors are rebuilt around Bravo even though the running machine still holds Alpha.

This is a direct source-level split between:

```
CURRENT REVIEW DESCRIPTOR = Bravo
RUNNING EXECUTION ACTOR    = Alpha
```

### 12. Alpha's registration actor autonomously wakes after commitment cooldown

Affected file:

```
packages/transaction-manager/src/machines/registration/registration.machine.ts
```

Frozen source:

```ts
1141 commitmentCooldown: {
1142   entry: ['logTransition'],
1143   invoke: {
1144     src: 'waitAfterCommitment',
...
1153     onDone: [
...
1160       { target: 'registeringDomain' },
1161     ],
```

For the Portal EOA path, when the cooldown finishes the machine transitions to:

```
registeringDomain
```

No new `START_REGISTRATION` event is required.

No new Portal "Register Bravo" button click is required.

No asynchronous Alpha callback has to race with Bravo.

The already-running Alpha actor simply continues.

### 13. `registeringDomain` explicitly feeds stale `context.name` into the final registration actor

Frozen source:

```ts
1379 registeringDomain: {
1380   entry: ['logTransition'],
1381   invoke: {
1382     src: 'submitRegistration',
1383     input: ({ context }) => ({
1384       name: context.name,
1385       commitment: context.commitment!,
1386       signer: context.signer!,
1387       duration: context.duration,
1388       selectedToken: context.selectedToken,
1389       owner: context.ownerAddress ?? context.accountAddress!,
1394       publicClient: context.publicClient!,
1396       resolverAddress: context.resolverAddress!,
1397       id: REGISTRATION_TX_IDS.register,
1398     }),
```

At the vulnerable point:

```
context.name = Alpha
Portal prop  = Bravo
```

### 14. The actual registrar calldata directly encodes stale Alpha

Affected file:

```
packages/transaction-manager/src/machines/registration/registration.actors.ts
```

Shared encoder:

```ts
223 export function encodeRegisterCall({
224   name,
...
237 }): { to: Address; data: Hex; value: bigint } {
238   return {
239     to: ENS_SEPOLIA_CONTRACTS.ETHRegistrar,
240     data: encodeFunctionData({
241       abi: ethRegistrarRegisterSnippet,
242       functionName: 'register',
243       args: [
244         name.replace('.eth', ''),
245         owner,
246         secret,
247         zeroAddress,
248         resolverAddress,
249         duration,
250         paymentToken,
251         REFERER_ADDRESS,
252       ],
253     }),
254     value: 0n,
255   }
256 }
```

Final submission:

```ts
924 const registerCall = encodeRegisterCall({
925   name: input.name,
926   owner: input.owner,
927   secret: input.commitment.secret,
928   duration: input.duration,
929   paymentToken: normalizedPaymentToken,
930   resolverAddress: input.resolverAddress,
931 })

933 const request = createTransactionRequest({
934   signer: input.signer,
935   from: accountAddress,
936   chainId: input.publicClient.chain?.id ?? sepolia.id,
937   calls: [registerCall],
938 })

940 const txId = transactionManager.startTransaction(
941   {
942     type: 'custom',
943     request,
944   },
945   input.signer,
946   {
947     id: input.id,
948     description: `Register ${input.name}.eth`,
949     publicClient: input.publicClient,
950     timeout: 120_000,
951   },
952 )
```

For the reproduced stale actor:

```
input.name = pxch.eth
```

so the actual request encodes:

```
register("pxch", ...)
```

### 15. `startTransaction()` immediately starts the prebuilt stale request

Affected file:

```
packages/transaction-manager/src/providers/transactionManager.ts
```

Frozen source:

```ts
225 const txId = transactionOptions.id || generateTransactionId()

227 // Create and start the transaction actor
228 const actor = createActor(transactionMachine, {
229   input: {
230     intent,
231     request,
232     signer,
233     publicClient,
234     options: transactionOptions,
235     chainId,
236     useSmartAccount,
237   },
238 })
...
251 actor.start()
```

This is not merely a stale preview object saved somewhere.

The transaction actor is started immediately.

### 16. A prebuilt request transitions directly into transaction submission

Affected file:

```
packages/transaction-manager/src/machines/transaction.machine.ts
```

Frozen source:

```ts
349 idle: {
350   always: [
...
357     {
358       // If we have a pre-prepared request, skip to submitting
359       guard: ({ context }) => !!context.request && !!context.publicClient,
360       target: 'submitting',
361     },
362   ],
...
427 submitting: {
428   invoke: {
429     src: 'submitTransaction',
430     input: ({ context }) => ({
431       request: context.request,
432       signer: context.signer,
433       options: context.options,
434       publicClient: context.publicClient,
435     }),
```

The Alpha request therefore proceeds to the wallet without requiring Portal to reconstruct it from the current Bravo route.

### 17. Downstream EOA validation checks the signer, but cannot correct the wrong ENS name

Affected file:

```
packages/transaction-manager/src/actors/eoa-transport.actor.ts
```

Frozen source:

```ts
35 // Verify the request's declared `from` matches the account the wallet will
...
39 const walletAddress = walletClient.account?.address
40 if (!walletAddress || !isAddressEqual(walletAddress, eoaRequest.from)) {
...
48 }

50 // Build transaction params
52 const txParams: any = {
53   account: eoaRequest.from,
54   to: eoaRequest.to,
55   value: eoaRequest.value,
56   data: eoaRequest.data,
...
70 return safeSendTransaction(walletClient, txParams)
```

This correctly prevents a sender/account mismatch.

It does **not** know that the Portal UI has switched from Alpha to Bravo, and it does not decode/rebind the ENS label.

So no downstream transaction validation repairs the stale resource.

### 18. Runtime directly captured the exact stale registrar label before authorization

The PoC wraps the headless wallet's `eth_sendTransaction` request and decodes calls sent to the real ENS registrar.

The successful run observed:

```
[AUDIT STALE REQUEST]
current-ui=portal-bravo-mtwxqxch.eth
current-total-usd=8.01
pending-register-label=pxch
wallet-send-pending=1
```

This is the decisive transaction-integrity boundary:

```
CURRENT APPLICATION RESOURCE = Bravo
ACTUAL PENDING REGISTER CALL  = Alpha
```

The PoC did not infer the label from UI text. It decoded the actual pending registrar calldata.

### 19. Authorizing the stale request causes real wrong-name chain state and the Alpha-sized debit

The same run then authorized the single pending wallet transaction and read direct local-chain state:

```
[AUDIT POST]
Alpha=pxch.eth available=false
Bravo=portal-bravo-mtwxqxch.eth available=true
current-Bravo-total-usd=8.01
actual-USDC-debit=160.109598
```

This establishes:

```
Alpha registration succeeded
Bravo was not registered
victim balance decreased by Alpha-sized amount
```

The result is authoritative contract/payment state, not presentation.

### 20. Success handling uses the current React `name`, creating a second source-level resource split

Affected file:

```
apps/portal/src/features/register/hooks/useRegistrationSuccessRedirect.ts
```

Frozen source:

```tsx
33 useEffect(() => {
34   if (hasRedirectedRef.current) return
35   if (!isSuccess) return
...
39   const redirectToProfile = () =>
40     navigate({
41       to: '/$name',
42       params: { name },
43       search: { registered: true, duration: durationSeconds, paid },
44       replace: true,
45     })
...
73 }, [isSuccess, name, durationSeconds, paid, navigate, queryClient])
```

If the stale Alpha actor reports success after the route has become Bravo, this hook consumes the **current Bravo prop**.

That is consistent with the same underlying divergence:

```
machine success resource = Alpha
current React resource   = Bravo
```

The core Critical impact does not rely on this redirect behavior, but it is additional evidence that registration state and route resource are not bound together.

***

## Complete Data Flow

```
/register?name=Alpha
        ↓
RegisterName(name=Alpha)
        ↓
useRegistrationTransactions(name=Alpha)
        ↓
useActorRef(registrationMachine, { input: { chainId } })
        ↓
START_REGISTRATION(name=Alpha)
        ↓
registrationMachine.context.name = Alpha
        ↓
resolver deployment
        ↓
Alpha commitment generated + submitted
        ↓
token approval (if needed)
        ↓
commitmentCooldown
        ↓
user closes modal
        ↓
Portal sidebar search navigates:
  /register?name=Alpha
          →
  /register?name=Bravo
        ↓
useBlocker warning accepted
        ↓
current React name = Bravo
current checkout   = Bravo
current total      = Bravo price
        ↓
NO CANCEL is sent to Alpha registration actor
        ↓
old actor still has:
  context.name       = Alpha
  context.commitment = Alpha commitment
        ↓
cooldown expires
        ↓
registeringDomain
        ↓
submitRegistration(name=context.name)
        ↓
encodeRegisterCall(name=Alpha)
        ↓
ETHRegistrar.register("Alpha", ...)
        ↓
transactionManager.startTransaction()
        ↓
transaction machine immediately submits request
        ↓
Portal still displays Bravo
        ↓
victim authorizes pending wallet transaction
        ↓
Alpha REGISTERED
Bravo AVAILABLE
Alpha-sized USDC debit
```

***

## Why This Is Not Merely a UI/UX Issue

The bug does not stop at a stale label.

The successful PoC decoded the **actual pending wallet calldata**:

```
pending-register-label=pxch
```

while the current Portal page was:

```
current-ui=portal-bravo-mtwxqxch.eth
```

After authorization, direct contract reads established:

```
Alpha available = false
Bravo available = true
```

and a direct ERC-20 balance read established:

```
actual USDC debit = 160.109598
```

Therefore the vulnerable consequence is real chain/payment state.

***

## Why This Is Not Intended Design

Portal itself warns:

> **"Your registration is in progress. Leaving may interrupt it and you could lose your commitment. Are you sure you want to leave?"**

That warning communicates that leaving may **interrupt** the registration and lose the commitment.

A design in which accepting that warning:

```
1. switches the application to Bravo;
2. displays Bravo's registration page;
3. keeps Alpha executing invisibly;
4. only afterward constructs a new real final register(Alpha) wallet transaction;
```

is inconsistent with the user-facing lifecycle semantics.

The `$160.11` Alpha versus `$8.01` Bravo price difference makes the consequence easier to see, but the lifecycle violation exists independently of that amount difference.

The intended invariant should be:

```
current Portal registration name
==
running registration actor name
==
name encoded in the next wallet transaction
```

or, at minimum:

```
if registration route/search name changes
then the previous registration actor must be cancelled or isolated
before any later wallet authorization can be generated
```

***

## Why Earlier Alpha Actions Do Not Make the Final Stale Transaction Safe

The user legitimately performed earlier Alpha setup operations, including the Alpha commitment and the token allowance required for Alpha's checkout. Those earlier operations are **not** claimed to be unauthorized.

That distinction matters.

The bug occurs only after the user explicitly leaves Alpha and Portal has transitioned to Bravo:

```
Alpha commitment + token approval completed while Alpha is current
        ↓
machine reaches commitmentCooldown
        ↓
user explicitly accepts Portal's "leave registration" warning
        ↓
Portal becomes Bravo
        ↓
Portal displays Bravo and Bravo's $8.01 checkout
        ↓
old Alpha actor is still alive
        ↓
only now does it create the final register(Alpha) wallet transaction
```

A triager can fairly argue that the user knowingly granted Alpha's allowance earlier. That weakens any attempt to frame the amount itself as a separate wrong-amount authorization bug.

It does **not** answer the wrong-name issue.

The final registrar calldata is a **new security-sensitive transaction generated after the application has moved away from Alpha**. Immediately before that transaction is authorized:

```
current Portal resource = Bravo
encoded registrar label = Alpha
```

The navigation warning also tells the user that leaving may interrupt the flow or lose the commitment. It does not disclose that Portal will keep Alpha running invisibly and later generate the final Alpha registration transaction while a different name is current.

Accordingly, this report relies on the post-switch **wrong-name transaction** as the Critical invariant violation. The Alpha-sized debit is supporting evidence of the consequence of executing that wrong transaction.

***

## Why the Amount Divergence Is Supporting Evidence, Not the Core Severity Claim

Portal's Alpha flow performs the token-approval step before the user abandons Alpha. The amount passed into that approval is derived from Alpha's saved checkout state:

```
savedParams.tokenPrice = Alpha price
```

and the registration steps progress through:

```
deployResolver
→ commit
→ approve
→ register
```

The successful PoC reaches `commitmentCooldown` only after the earlier Alpha approval has completed.

Therefore this report does **not** argue that the victim first authorizes Alpha's `160.109598 USDC` amount while looking at Bravo. A reviewer could correctly say that the allowance for the Alpha amount was originally granted while Alpha and its `$160.11` checkout were visible.

That does not repair the security bug.

The decisive transaction-integrity violation happens later:

```
Alpha approval/commitment already exists
        ↓
user accepts Portal's "leave registration" warning
        ↓
Portal transitions to Bravo
        ↓
current page/checkout = Bravo
        ↓
old Alpha actor later creates NEW registrar calldata
        ↓
pending wallet transaction = register(Alpha, ...)
```

The new post-switch security-sensitive object is the registrar transaction itself. Its encoded ENS label is Alpha even though the active Portal registration resource is Bravo.

The price spread is therefore used only to show that this stale-resource execution is materially consequential:

```
abandoned Alpha checkout = $160.11
current Bravo checkout   = $8.01
actual stale Alpha debit = 160.109598 USDC
```

The Critical mapping stands on the **wrong ENS name in the post-switch wallet transaction**, even if the amount divergence is treated only as impact evidence.

***

## Why the Wallet Authorization Does Not Repair the Application Error

The final EOA request contains encoded registrar calldata.

The transaction manager validates that the connected wallet's address matches `request.from`, which is correct and unrelated to the stale name.

No Portal layer revalidates:

```
encoded ENS label == currently displayed ENS name
```

before sending the request.

The application therefore reaches the wallet boundary with already-divergent security-sensitive transaction data.

A sophisticated wallet may expose raw calldata or decode the method; that does not remove the application vulnerability. The scoped impact concerns the application presenting/pre-filling incorrect transaction details and causing the user to submit an unintended transaction. Portal itself generated the stale calldata after it had transitioned to Bravo.

***

## Why the PoC Does Not Manufacture the Bug

The PoC does **not**:

* delay or forge an RPC response;
* modify registrar responses;
* edit XState/React internals;
* change application state through devtools;
* substitute calldata;
* alter resolver state;
* change the wallet request;
* require a malicious RPC;
* require an attacker-controlled backend;
* require a race between Alpha and Bravo network requests.

It simply:

1. runs Alpha through Portal's ordinary production registration flow;
2. waits until the real Alpha actor reaches the real commitment cooldown;
3. closes the modal through the UI;
4. navigates through Portal's own search UI to Bravo;
5. accepts Portal's real navigation warning;
6. waits for the existing Alpha cooldown to expire;
7. inspects the real wallet request generated by production code;
8. authorizes it;
9. reads real local-chain post-state.

The stale continuation is therefore deterministic from application lifecycle behavior.

***

## `E2E_MOCK_INDEXER=true` Does Not Mock the Security-Sensitive Result

The local E2E run uses:

```
E2E_MOCK_INDEXER=true
```

The decisive assertions in this PoC are not derived from an indexer response.

The PoC directly performs local-chain reads against:

* the ENS registrar's `isAvailable(label)`;
* the USDC token's `balanceOf(owner)`.

The critical transaction is the real production `eth_sendTransaction` request produced by Portal and decoded before authorization.

The final state:

```
Alpha available = false
Bravo available = true
victim USDC decreased by 160.109598
```

therefore comes from authoritative local contract state.

An indexer mock cannot create the decoded stale registrar label, make the registrar transaction succeed, or debit the ERC-20 balance.

***

## Why This Is Different From the Previously Reported Portal Stale-Recipient Transfer

The previously reported Portal transfer issue has a different mechanism:

```
same transfer flow
recipient A captured during async preparation
user edits recipient to B
old preparation completes
transfer calldata still targets A
```

This registration issue is:

```
registration actor snapshots name A
commitment for A succeeds
user leaves the entire A registration context
Portal changes to name B
actor A is never cancelled
cooldown autonomously completes
registrar calldata still targets A
```

Different vulnerable state:

```
transfer recipient closure
vs
registration actor resource/lifecycle
```

Different transaction:

```
name transfer
vs
new ENS registration
```

Different fix:

```
revalidate recipient after async preparation
vs
cancel/rekey registration actor on resource change
```

***

## Why This Is Not One of the Disclosed Known Issues

This report is not:

* `EXP-4337-002` generic missing chainId validation;
* `EXP-4337-003` generic `request.from` mismatch;
* `SEC-TXM-002` generic cached smart-account address mismatch;
* `R2-03` broad 7-day session-key scope;
* `R2-06` local-only session revoke;
* `R3-02` stopped actor awaits forever;
* `R3-03` registration poller hang;
* `R3-04` commitment retry wrong step;
* `R3-05` localStorage active/history collision;
* `R3-06` archived chainId;
* `R3-07` reused transaction-ID bookkeeping;
* `R3-08` stale failure on success;
* QA price display inconsistency;
* bulk-renew duplicate payment;
* Portal stale-recipient transfer;

The root invariant here is:

> **An in-progress Portal registration actor must not survive a selected-name change and later submit the old name after the user has explicitly left that registration and the application is presenting a different name.**

***

## Audited Revision Verification

ENS Audit Competition frozen import:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Upstream audited revision:

```
63772fd872af472ced58b009499355f3430c2a86
```

Frozen archive SHA-256:

```
c7fead4607ae2e263b9288a8980c80288fcfe0a928616783302ce362edead122
```

Runtime environment:

```
Node:       v22.22.1
chain:      11155111 (Sepolia local E2E / Anvil)
Anvil RPC:  http://127.0.0.1:8545
Portal:     http://127.0.0.1:3001
```

The browser E2E completed with:

```
1 passed (1.3m)
POC RETURN CODE: 0
```

***

## Impact Details

The successful run used dynamically generated names:

```
Alpha:
pxch.eth

Bravo:
portal-bravo-mtwxqxch.eth

victim:
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

### Authoritative pre-state

```
Alpha available = true
Bravo available = true
```

The test explicitly asserted both against the local registrar before starting.

### Alpha before abandonment

```
Alpha current                 = pxch.eth
Alpha displayed total         = $160.11
real commitment flow completed
machine state                 = commitmentCooldown
final register transaction    = not yet sent
Alpha available               = true
```

Observed:

```
[AUDIT PRICE A] current=pxch.eth; displayed-total-usd=160.11
[AUDIT COOLDOWN] Ready in 60s
```

### Explicit abandonment / navigation

Portal displayed the built-in warning:

```
[AUDIT BLOCKER] Your registration is in progress. Leaving may interrupt it and you could lose your commitment. Are you sure you want to leave?
```

The user accepted the warning and Portal navigated through its own SPA search to Bravo.

### Current Bravo state

```
current name           = portal-bravo-mtwxqxch.eth
current displayed total = $8.01
abandoned Alpha total   = $160.11
```

Observed:

```
[AUDIT SWITCH] current=portal-bravo-mtwxqxch.eth; Bravo-total-usd=8.01; abandoned-Alpha-total-usd=160.11
```

### Post-switch wallet transaction

After the old Alpha cooldown expired:

```
current application name = Bravo
current displayed total   = $8.01
pending wallet sends      = 1
actual registrar label    = pxch (Alpha)
```

Observed:

```
[AUDIT STALE REQUEST] current-ui=portal-bravo-mtwxqxch.eth; current-total-usd=8.01; pending-register-label=pxch; wallet-send-pending=1
```

The PoC decodes the actual pending `eth_sendTransaction` calldata directed to the ENS registrar and verifies that the call is:

```
register("pxch", ...)
```

not:

```
register("portal-bravo-mtwxqxch", ...)
```

### Authoritative post-state

After the user authorizes the pending request:

```
Alpha available            = false
Bravo available            = true
current Bravo displayed    = $8.01
actual victim USDC debit   = 160.109598
```

Observed:

```
[AUDIT POST] Alpha=pxch.eth available=false; Bravo=portal-bravo-mtwxqxch.eth available=true; current-Bravo-total-usd=8.01; actual-USDC-debit=160.109598
```

The mismatch at the wallet-authorization boundary is:

```
IMMEDIATELY BEFORE AUTHORIZATION
--------------------------------
current Portal resource     = Bravo

PENDING WALLET TRANSACTION
--------------------------
registrar method            = register(...)
encoded label               = Alpha

AUTHORITATIVE FINAL STATE
-------------------------
Alpha                       = REGISTERED
Bravo                       = AVAILABLE
```

Material consequence observed in the same run:

```
abandoned Alpha total       = $160.11
current Bravo total         = $8.01
actual stale Alpha debit    = 160.109598 USDC
```

***

## Preconditions

1. The user uses Portal's EOA registration flow.
2. Alpha is available and the user starts registering it.
3. The Alpha flow progresses through resolver deployment, commitment, and any required token approval.
4. Alpha reaches the normal commitment cooldown.
5. Before the cooldown ends, the user closes the transaction modal.
6. The user uses Portal's own search UI to select a different available name, Bravo.
7. Portal displays its "leaving may interrupt registration / lose your commitment" warning and the user accepts it.
8. Alpha remains available until its cooldown completes.
9. The user authorizes the later wallet transaction generated while Portal is already displaying Bravo.

No:

* malicious RPC;
* response corruption;
* compromised wallet;
* application administrator;
* ENS governance actor;
* browser extension;
* direct contract interaction;
* devtools state editing;
* network-response race manipulation

is required.

***

## Recommended Remediation

### Option 1 — bind the Portal registration actor lifetime to `name`

The strongest structural fix is to ensure an actor created for Alpha cannot survive as the active registration actor after the selected name becomes Bravo.

For example, move the actor into a component keyed by `name`, or otherwise recreate it when the registration resource changes.

Conceptually:

```tsx
<RegisterName key={name} name={name} />
```

or key a dedicated registration provider/actor boundary rather than relying only on prop updates.

The exact implementation should be chosen carefully so that intentional route updates cannot preserve stale machine state.

### Option 2 — explicitly cancel registration when `name` changes

`useRegistrationTransactions` already has the primitive:

```tsx
actor.send({ type: 'CANCEL' })
closeModal()
clearTransaction()
```

Use it when the registration resource changes during an active flow.

The invariant should be:

```
oldName != newName && registration in progress
    =>
cancel old actor before new resource is rendered/authorized
```

### Option 3 — make the route blocker perform actual cancellation when the user confirms leaving

Currently the blocker only asks for confirmation:

```
shouldLeave = confirm(...)
return !shouldLeave
```

When the user explicitly accepts the warning, Portal should terminate the abandoned registration actor or require the user to deliberately choose between:

* continue Alpha;
* cancel Alpha and leave.

Do not let "leave Alpha" mean "hide Alpha while Alpha continues to transaction submission."

### Option 4 — validate the current registration resource before final transaction construction

Before `registeringDomain` submits the reveal, fail closed unless the actor's registered resource is still the application's active resource.

For example, a higher-level integration could verify:

```
registrationActor.context.name
==
current Portal /register search name
```

If they differ, cancel the flow and require a fresh explicit review.

This is a defense-in-depth check; actor lifecycle should still be fixed.

### Option 5 — do not derive review descriptors from one resource while executing another

Portal's transaction descriptors are rebuilt from the live React `name` while the actual machine execution uses `context.name`.

During an active registration, the review UI should derive its authoritative resource from the same immutable registration object that will be submitted, or cancel the old object on route change.

The invariant must be:

```
review transaction name
==
encoded transaction name
```

### Option 6 — make success routing consume the completed registration's resource

`useRegistrationSuccessRedirect()` currently receives the current React `name`.

Success state should carry the actually completed name so a stale Alpha completion cannot be presented as Bravo's registration success.

### Option 7 — add the exact Alpha → cooldown → Bravo regression test

At minimum:

```
start Alpha registration
→ reach commitmentCooldown
→ confirm final register tx has not yet been sent
→ close transaction modal
→ use Portal search to switch to Bravo
→ accept "leave registration" warning
→ assert Portal shows Bravo
→ wait for old Alpha cooldown
```

Expected fixed behavior:

```
no pending registrar transaction for Alpha
old Alpha actor is cancelled/stopped
Alpha remains available
Bravo remains current
only a fresh Bravo registration can later create a registrar transaction
```

### Recommended invariant

> **Once Portal accepts a change from registration resource Alpha to resource Bravo, no subsequent transaction created by the abandoned Alpha actor may reach the connected wallet. The name displayed by Portal at authorization time must be the same name encoded into the registrar transaction.**

***

## References

### In-scope Portal

* `apps/portal/src/routes/register/index.tsx`
  * stores selected name in `/register/` search state;
  * renders `RegisterName name={name}`.
* `apps/portal/src/features/dashboard/components/HomeSearchInput.tsx`
  * performs SPA navigation to `/register` with a new `name` search parameter.
* `apps/portal/src/features/register/components/RegisterName.tsx`
  * invokes `useRegistrationTransactions({ name, duration })`;
  * displays current `name`;
  * displays current-name checkout;
  * prompts the user about leaving an in-progress registration;
  * does not cancel the actor when navigation is accepted.
* `apps/portal/src/features/register/components/RegisterNameSummary.tsx`
  * queries and renders pricing from the current `name` prop.
* `apps/portal/src/features/register/hooks/useRegistrationTransactions.ts`
  * creates the actor with `input: { chainId }`;
  * sends `START_REGISTRATION` with the current name only at flow start;
  * exposes a cancellation helper but does not bind it to name navigation;
  * builds current transaction descriptors from the live React name.
* `apps/portal/src/features/transaction-manager/hooks/useTransactionModal.ts`
  * route reset only closes the modal.
* `apps/portal/src/features/transaction-manager/components/TransactionModal.tsx`
  * modal close does not clear non-failed registration state.
* `apps/portal/src/features/register/hooks/useRegistrationSuccessRedirect.ts`
  * redirects based on the current React name after machine success.

### In-scope transaction-manager

* `packages/transaction-manager/src/machines/registration/registration.machine.ts`
  * snapshots `START_REGISTRATION.name`;
  * creates commitment from `context.name`;
  * automatically advances from `commitmentCooldown`;
  * submits final registration with `name: context.name`.
* `packages/transaction-manager/src/machines/registration/registration.actors.ts`
  * `encodeRegisterCall()` encodes the supplied name as the first `register()` argument;
  * `submitRegistrationActor()` creates the request and calls `transactionManager.startTransaction()`.
* `packages/transaction-manager/src/providers/transactionManager.ts`
  * creates and immediately starts the transaction actor.
* `packages/transaction-manager/src/machines/transaction.machine.ts`
  * prebuilt requests go directly from `idle` to `submitting`.
* `packages/transaction-manager/src/actors/eoa-transport.actor.ts`
  * validates signer/from equality but submits the already-prepared calldata unchanged.

## Proof of Concept

### PoC Classification

I implemented and executed a repo-native **browser-to-chain Portal E2E PoC** against the frozen competition source.

The successful run uses the real:

* Portal registration UI;
* `/register?name=` search-parameter transition;
* production `useRegistrationTransactions` hook;
* production registration XState machine;
* real resolver/commit/approval progression;
* real `commitmentCooldown`;
* production transaction-manager;
* production EOA transaction transport;
* headless wallet;
* local Sepolia/Anvil registrar;
* actual `ETHRegistrar.register()` calldata;
* direct registrar availability reads;
* direct ERC-20 balance reads.

The test does **not**:

* delay RPC responses;
* reorder honest network responses;
* forge availability data;
* mutate application/XState state through devtools;
* replace calldata;
* modify the wallet request;
* require a malicious RPC.

It reproduces the issue only through Portal's ordinary registration UI and the application's own navigation behavior.

### Exact executed PoC artifact

The exact file that produced the transcript below is:

```
e2e/projects/portal/tests/portal-stale-registration-name-final.spec.ts
```

Artifact identity:

```
line count: 966
SHA-256:
0c5ded13015f25fe6b8fb7634e9c9f1622efc584168a4a342cfa7b6e9b6bbcd1
```

Playwright reported the test at:

```
projects/portal/tests/portal-stale-registration-name-final.spec.ts:398:5
```

The file was hashed and counted both before and after the run:

```
Pre-run lines : 966
Post-run lines: 966

Pre-run SHA   : 0c5ded13015f25fe6b8fb7634e9c9f1622efc584168a4a342cfa7b6e9b6bbcd1
Post-run SHA  : 0c5ded13015f25fe6b8fb7634e9c9f1622efc584168a4a342cfa7b6e9b6bbcd1

SOURCE IMMUTABILITY PASS
```

Because this submission will not include a separate PoC attachment, the **entire executed PoC is reproduced inline below**.

The contents inside the TypeScript code fence are **byte-for-byte identical** to the executed 966-line file identified above. Extracting the code fence contents with the final newline preserved yields:

```
line count: 966
SHA-256: 0c5ded13015f25fe6b8fb7634e9c9f1622efc584168a4a342cfa7b6e9b6bbcd1
```

### Complete executed PoC — byte-identical inline copy

```ts
/**
 * ENS Portal stale-registration actor PoC
 *
 * Core Critical invariant:
 *
 *   CURRENT PORTAL RESOURCE = Bravo
 *   PENDING ETHRegistrar.register() LABEL = abandoned Alpha
 *
 * The Alpha/Bravo price difference and final debit are supporting
 * material-impact evidence. The core vulnerability is the post-switch
 * wrong-name registrar transaction.
 */

import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  type Web3ProviderBackend,
  Web3RequestKind,
} from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import { decodeFunctionData, parseAbi } from 'viem'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL =
  process.env.PORTAL_APP_URL ?? 'http://localhost:3001'

const suffix = Date.now().toString(36)

// Keep Alpha exactly four characters so it is materially more expensive.
const ALPHA = `p${suffix.slice(-3)}`

// Bravo is deliberately long / cheaper.
const BRAVO = `portal-bravo-${suffix}`

const ALPHA_NAME = `${ALPHA}.eth`
const BRAVO_NAME = `${BRAVO}.eth`

const ensjsSepolia = ensL1Contracts[supportedL1Chains.sepolia]

const ETH_REGISTRAR =
  ensjsSepolia.ensEthRegistrar.address

const USDC =
  ensjsSepolia.usdc.address

const registrarAbi = parseAbi([
  'function isAvailable(string label) view returns (bool)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
])

const erc20Abi = parseAbi([
  'function balanceOf(address owner) view returns (uint256)',
])

type AuditRpcRequest = {
  method: string
  params?: unknown
}

/**
 * Decode only wallet eth_sendTransaction calls that actually target
 * the ENS registrar's register() method.
 */
function getRegisterLabel(
  request: AuditRpcRequest,
): string | undefined {
  if (
    request.method !==
    Web3RequestKind.SendTransaction
  ) {
    return undefined
  }

  const params =
    request.params as
      | readonly unknown[]
      | undefined

  const tx = params?.[0] as
    | {
        to?: string
        data?: `0x${string}`
        input?: `0x${string}`
      }
    | undefined

  const data =
    tx?.data ?? tx?.input

  if (
    !tx?.to ||
    !data
  ) {
    return undefined
  }

  if (
    tx.to.toLowerCase() !==
    ETH_REGISTRAR.toLowerCase()
  ) {
    return undefined
  }

  try {
    const decoded =
      decodeFunctionData({
        abi: registrarAbi,
        data,
      })

    if (
      decoded.functionName !==
      'register'
    ) {
      return undefined
    }

    return String(
      decoded.args[0],
    )
  } catch {
    return undefined
  }
}

async function isAvailable(
  label: string,
): Promise<boolean> {
  return publicClient.readContract({
    address: ETH_REGISTRAR,
    abi: registrarAbi,
    functionName: 'isAvailable',
    args: [label],
  })
}

async function usdcBalance(
  owner: `0x${string}`,
): Promise<bigint> {
  return publicClient.readContract({
    address: USDC,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [owner],
  })
}

async function readDisplayedTotalUsd(
  page: Page,
): Promise<number> {
  const checkout =
    page.locator(
      'section[aria-label="Checkout summary"]',
    )

  const totalLabel =
    checkout.getByText(
      'Total',
      { exact: true },
    )

  await expect(
    totalLabel,
  ).toBeVisible({
    timeout: 30_000,
  })

  let total = 0

  await expect
    .poll(
      async () => {
        const text =
          (
            await totalLabel
              .locator('xpath=..')
              .innerText()
          ).replace(/,/g, '')

        const match =
          text.match(
            /\$([0-9]+(?:\.[0-9]+)?)/,
          )

        total = match
          ? Number(match[1])
          : 0

        return total
      },
      {
        timeout: 30_000,
      },
    )
    .toBeGreaterThan(0)

  return total
}

/**
 * Progress the real Alpha registration through:
 *
 *   deploy resolver
 *   → commitment
 *   → ERC-20 approval if required
 *   → commitmentCooldown
 *
 * Stop BEFORE the final ETHRegistrar.register() transaction.
 */
async function driveAlphaToCooldown(
  page: Page,
  wallet: Web3ProviderBackend,
): Promise<void> {
  const transactionDialog =
    page.locator(
      '[data-slot="dialog-content"]',
    )

  await expect(
    transactionDialog,
  ).toBeVisible({
    timeout: 30_000,
  })

  const startButton =
    transactionDialog.getByRole(
      'button',
      {
        name: /^Start$/i,
      },
    )

  await startButton.waitFor({
    state: 'visible',
    timeout: 30_000,
  })

  await startButton.click()

  await expect(
    transactionDialog.getByText(
      'Transaction flow',
    ),
  ).toBeVisible({
    timeout: 30_000,
  })

  const deadline =
    Date.now() + 180_000

  while (
    Date.now() < deadline
  ) {
    /**
     * When "Ready in Ns" appears, all earlier steps have
     * completed and Register is waiting for commitment age.
     */
    const cooldown =
      transactionDialog.getByText(
        /^Ready in \d+s$/,
      )

    if (
      await cooldown
        .isVisible()
        .catch(() => false)
    ) {
      console.log(
        `[AUDIT COOLDOWN] ${await cooldown.innerText()}`,
      )

      return
    }

    /**
     * Normal explicit wallet action.
     */
    const openWalletButton =
      transactionDialog.getByRole(
        'button',
        {
          name: /open wallet/i,
        },
      )

    if (
      await openWalletButton
        .isVisible()
        .catch(() => false)
    ) {
      await openWalletButton.click()

      await authorizeTransaction(
        wallet,
        60_000,
      )

      await page.waitForTimeout(
        400,
      )

      continue
    }

    /**
     * Some Portal states expose the wallet action as an
     * icon immediately before a "Waiting..." button.
     */
    const waitingButton =
      transactionDialog.getByRole(
        'button',
        {
          name: /^Waiting\.\.\.$/i,
        },
      )

    if (
      await waitingButton
        .isVisible()
        .catch(() => false)
    ) {
      const iconWalletButton =
        waitingButton.locator(
          'xpath=preceding-sibling::button[1]',
        )

      if (
        await iconWalletButton
          .isVisible()
          .catch(() => false)
      ) {
        await iconWalletButton.click()

        await authorizeTransaction(
          wallet,
          60_000,
        )

        await page.waitForTimeout(
          400,
        )

        continue
      }
    }

    const nextButton =
      transactionDialog.getByRole(
        'button',
        {
          name: /^(Start|Next)$/i,
        },
      )

    if (
      (
        await nextButton
          .isVisible()
          .catch(() => false)
      ) &&
      (
        await nextButton
          .isEnabled()
          .catch(() => false)
      )
    ) {
      await nextButton
        .click({
          timeout: 2_000,
        })
        .catch(() => {})

      await page.waitForTimeout(
        300,
      )

      continue
    }

    await page.waitForTimeout(
      250,
    )
  }

  throw new Error(
    'Timed out before Alpha reached commitment cooldown',
  )
}

test.describe(
  'AUDIT PoC - Portal stale registration actor',
  () => {
    test(
      'shows Bravo while stale Alpha actor submits register(Alpha)',
      async ({
        portalPage: page,
        wallet,
        accounts,
      }) => {
        test.setTimeout(
          420_000,
        )

        /**
         * Record the REAL wallet transaction requests produced
         * by Portal. Nothing is modified.
         */
        const sendRequests:
          AuditRpcRequest[] = []

        const originalRequest =
          wallet.request.bind(
            wallet,
          ) as (
            request: AuditRpcRequest,
          ) => Promise<unknown>

        wallet.request = (
          async (
            request: AuditRpcRequest,
          ) => {
            if (
              request.method ===
              Web3RequestKind.SendTransaction
            ) {
              sendRequests.push(
                request,
              )
            }

            return originalRequest(
              request,
            )
          }
        ) as Web3ProviderBackend['request']

        await connectWithHeadlessWallet(
          page,
          wallet,
        )

        const walletAddress =
          accounts.getAddress(
            'user',
          )

        const [
          alphaAvailableBefore,
          bravoAvailableBefore,
        ] = await Promise.all([
          isAvailable(ALPHA),
          isAvailable(BRAVO),
        ])

        expect(
          alphaAvailableBefore,
        ).toBe(true)

        expect(
          bravoAvailableBefore,
        ).toBe(true)

        console.log(
          `[AUDIT PRE] Alpha=${ALPHA_NAME}; Bravo=${BRAVO_NAME}; bothAvailable=true; wallet=${walletAddress}`,
        )

        /**
         * Start legitimate Alpha registration.
         */
        await page.goto(
          `${PORTAL_APP_URL}/register?name=${encodeURIComponent(
            ALPHA_NAME,
          )}`,
        )

        await expect(
          page
            .getByText(
              ALPHA_NAME,
              { exact: true },
            )
            .first(),
        ).toBeVisible({
          timeout: 30_000,
        })

        const alphaDisplayedTotalUsd =
          await readDisplayedTotalUsd(
            page,
          )

        console.log(
          `[AUDIT PRICE A] current=${ALPHA_NAME}; displayed-total-usd=${alphaDisplayedTotalUsd}`,
        )

        const paymentSection =
          page.locator(
            'section:has-text("Select payment method")',
          )

        await expect(
          paymentSection,
        ).toBeVisible({
          timeout: 20_000,
        })

        await paymentSection
          .getByRole(
            'button',
            {
              name: 'USDC',
            },
          )
          .first()
          .click()

        await paymentSection
          .getByRole(
            'button',
            {
              name: /^Register$/i,
            },
          )
          .click()

        /**
         * Complete everything up to commitment cooldown,
         * but do not send the final registrar transaction.
         */
        await driveAlphaToCooldown(
          page,
          wallet,
        )

        /**
         * CRITICAL PRECONDITION:
         * no ETHRegistrar.register() request exists yet.
         */
        const registerLabelsBeforeSwitch =
          sendRequests
            .map(
              getRegisterLabel,
            )
            .filter(
              (
                label,
              ): label is string =>
                Boolean(label),
            )

        expect(
          registerLabelsBeforeSwitch,
        ).toHaveLength(0)

        expect(
          await isAvailable(
            ALPHA,
          ),
        ).toBe(true)

        console.log(
          `[AUDIT PRE-SWITCH] Alpha=${ALPHA_NAME}; registrar-register-requests=${registerLabelsBeforeSwitch.length}; AlphaAvailable=true`,
        )

        /**
         * Close only the modal.
         *
         * This does NOT cancel the registration actor.
         */
        const transactionDialog =
          page.locator(
            '[data-slot="dialog-content"]',
          )

        await transactionDialog
          .getByRole(
            'button',
            {
              name: 'Close',
            },
          )
          .click()

        await expect(
          transactionDialog,
        ).toBeHidden({
          timeout: 10_000,
        })

        /**
         * Navigate to Bravo using Portal's own SPA search.
         *
         * Do NOT use page.goto() here because that would perform
         * a full page load and destroy the actor.
         */
        const searchInput =
          page
            .locator(
              'input[placeholder="Search..."]',
            )
            .first()

        await expect(
          searchInput,
        ).toBeVisible({
          timeout: 10_000,
        })

        await searchInput.fill(
          BRAVO,
        )

        const bravoOption =
          page
            .getByRole(
              'option',
            )
            .filter({
              hasText:
                BRAVO_NAME,
            })
            .first()

        await expect(
          bravoOption,
        ).toContainText(
          'Available to register',
          {
            timeout: 30_000,
          },
        )

        let blockerMessage = ''

        page.once(
          'dialog',
          async (
            dialog,
          ) => {
            blockerMessage =
              dialog.message()

            console.log(
              `[AUDIT BLOCKER] ${blockerMessage}`,
            )

            await dialog.accept()
          },
        )

        await bravoOption
          .getByText(
            BRAVO_NAME,
            {
              exact: true,
            },
          )
          .click()

        await page.waitForURL(
          (url) =>
            url.pathname ===
              '/register' &&
            url.searchParams.get(
              'name',
            ) ===
              BRAVO_NAME,
          {
            timeout: 30_000,
          },
        )

        await expect(
          page
            .getByText(
              BRAVO_NAME,
              {
                exact: true,
              },
            )
            .first(),
        ).toBeVisible({
          timeout: 30_000,
        })

        /**
         * Confirm that this was the actual in-progress
         * registration warning, not some unrelated browser dialog.
         */
        await expect
          .poll(
            () =>
              blockerMessage,
            {
              timeout: 10_000,
            },
          )
          .toContain(
            'Leaving may interrupt it and you could lose your commitment',
          )

        /**
         * Portal review context is now Bravo.
         */
        const bravoDisplayedTotalUsd =
          await readDisplayedTotalUsd(
            page,
          )

        expect(
          alphaDisplayedTotalUsd,
        ).toBeGreaterThan(
          bravoDisplayedTotalUsd +
            10,
        )

        console.log(
          `[AUDIT SWITCH] current=${BRAVO_NAME}; Bravo-total-usd=${bravoDisplayedTotalUsd}; abandoned-Alpha-total-usd=${alphaDisplayedTotalUsd}`,
        )

        /**
         * Alpha's existing actor remains alive.
         *
         * When commitmentCooldown expires, it autonomously
         * produces the final registrar request.
         */
        await expect
          .poll(
            () => {
              const labels =
                sendRequests
                  .map(
                    getRegisterLabel,
                  )
                  .filter(
                    (
                      label,
                    ): label is string =>
                      Boolean(
                        label,
                      ),
                  )

              return labels.at(
                -1,
              )
            },
            {
              timeout: 90_000,
            },
          )
          .toBe(
            ALPHA,
          )

        /**
         * CRITICAL INVARIANT:
         *
         * Current Portal resource == Bravo.
         */
        const currentUrl =
          new URL(
            page.url(),
          )

        expect(
          currentUrl.pathname,
        ).toBe(
          '/register',
        )

        expect(
          currentUrl.searchParams.get(
            'name',
          ),
        ).toBe(
          BRAVO_NAME,
        )

        await expect(
          page
            .getByText(
              BRAVO_NAME,
              {
                exact: true,
              },
            )
            .first(),
        ).toBeVisible()

        /**
         * Yet a real wallet sendTransaction request exists.
         */
        const pendingSendCount =
          wallet.getPendingRequestCount(
            Web3RequestKind.SendTransaction,
          )

        expect(
          pendingSendCount,
        ).toBeGreaterThanOrEqual(
          1,
        )

        /**
         * Decode the real pending registrar transaction.
         */
        const registerLabels =
          sendRequests
            .map(
              getRegisterLabel,
            )
            .filter(
              (
                label,
              ): label is string =>
                Boolean(label),
            )

        expect(
          registerLabels.at(
            -1,
          ),
        ).toBe(
          ALPHA,
        )

        expect(
          registerLabels.at(
            -1,
          ),
        ).not.toBe(
          BRAVO,
        )

        /**
         * Capture the balance immediately before the FINAL
         * post-switch registrar authorization.
         */
        const balanceBeforeFinalAuthorization =
          await usdcBalance(
            walletAddress,
          )

        console.log(
          `[AUDIT STALE REQUEST] current-ui=${BRAVO_NAME}; current-total-usd=${bravoDisplayedTotalUsd}; pending-register-label=${registerLabels.at(-1)}; wallet-send-pending=${pendingSendCount}`,
        )

        /**
         * Authorize exactly the wallet request Portal generated
         * while Bravo is the current registration resource.
         */
        await authorizeTransaction(
          wallet,
          60_000,
        )

        /**
         * Wait for authoritative registrar post-state.
         */
        await expect
          .poll(
            () =>
              isAvailable(
                ALPHA,
              ),
            {
              timeout: 120_000,
            },
          )
          .toBe(false)

        const [
          alphaAvailableAfter,
          bravoAvailableAfter,
          balanceAfter,
        ] =
          await Promise.all([
            isAvailable(ALPHA),
            isAvailable(BRAVO),
            usdcBalance(
              walletAddress,
            ),
          ])

        /**
         * Authoritative wrong-name result.
         */
        expect(
          alphaAvailableAfter,
        ).toBe(false)

        expect(
          bravoAvailableAfter,
        ).toBe(true)

        /**
         * Supporting material-impact evidence.
         */
        expect(
          balanceAfter,
        ).toBeLessThan(
          balanceBeforeFinalAuthorization,
        )

        const debitRaw =
          balanceBeforeFinalAuthorization -
          balanceAfter

        const debitUsd =
          Number(debitRaw) /
          1_000_000

        expect(
          debitUsd,
        ).toBeGreaterThan(
          bravoDisplayedTotalUsd +
            10,
        )

        console.log(
          `[AUDIT POST] Alpha=${ALPHA_NAME} available=${alphaAvailableAfter}; Bravo=${BRAVO_NAME} available=${bravoAvailableAfter}; current-Bravo-total-usd=${bravoDisplayedTotalUsd}; actual-USDC-debit=${debitUsd.toFixed(6)}`,
        )

        console.log(
          '============================================================',
        )
        console.log(
          'CRITICAL PORTAL STALE REGISTRATION CONFIRMED',
        )
        console.log(
          '============================================================',
        )
        console.log(
          `Abandoned Alpha: ${ALPHA_NAME}`,
        )
        console.log(
          `Current Portal name: ${BRAVO_NAME}`,
        )
        console.log(
          `Current Bravo displayed total USD: ${bravoDisplayedTotalUsd}`,
        )
        console.log(
          `Pending/approved registrar label: ${ALPHA}`,
        )
        console.log(
          `Alpha registered: ${!alphaAvailableAfter}`,
        )
        console.log(
          `Bravo still available: ${bravoAvailableAfter}`,
        )
        console.log(
          `Actual USDC debit: ${debitUsd.toFixed(6)}`,
        )
        console.log(
          '============================================================',
        )
      },
    )
  },
)
```

***

## Commands

### Terminal 1 — E2E infrastructure

```bash
cd /workspaces/ens/audit-comp-ens

nvm use 22
corepack enable

export PATH="$HOME/.foundry/bin:$PATH"
export ANVIL_RPC_URL=http://127.0.0.1:8545
export E2E_MOCK_INDEXER=true

set +e
set +o errexit

pnpm e2e:infra:up

docker compose -f e2e/infra/docker-compose.yml ps

curl -sS \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  http://127.0.0.1:8545
```

Expected chain:

```
0xaa36a7
```

which is Sepolia chain ID `11155111`.

### Terminal 2 — Portal

```bash
cd /workspaces/ens/audit-comp-ens

nvm use 22
corepack enable

cp -f apps/portal/.env.ci apps/portal/.env.local
cp -f e2e/.env.ci e2e/.env

export ANVIL_RPC_URL=http://127.0.0.1:8545
export PORTAL_APP_URL=http://127.0.0.1:3001
export E2E_MOCK_INDEXER=true

pnpm --filter portal exec vite dev \
  --host 0.0.0.0 \
  --port 3001 \
  --strictPort
```

### Terminal 3 — execute the exact PoC artifact

```bash
cd /workspaces/ens/audit-comp-ens/e2e

nvm use 22
corepack enable

set +e
set +o errexit
set -o pipefail

export ANVIL_RPC_URL=http://127.0.0.1:8545
export PORTAL_APP_URL=http://127.0.0.1:3001
export E2E_MOCK_INDEXER=true
export TZ=UTC

POC_FILE=projects/portal/tests/portal-stale-registration-name-final.spec.ts
LOG_FILE=/tmp/portal-stale-registration-name-final.log

PRE_LINES=$(wc -l < "$POC_FILE")
PRE_SHA=$(sha256sum "$POC_FILE" | awk '{print $1}')

echo "Pre-run lines : $PRE_LINES"
echo "Pre-run SHA   : $PRE_SHA"

rm -f "$LOG_FILE"

node_modules/.bin/playwright test \
  "$POC_FILE" \
  --config=projects/portal/playwright.config.ts \
  --project=portal-e2e \
  --workers=1 \
  --retries=0 \
  --trace=on \
  --reporter=line \
  2>&1 | tee "$LOG_FILE"

POC_RC=${PIPESTATUS[0]}

POST_LINES=$(wc -l < "$POC_FILE")
POST_SHA=$(sha256sum "$POC_FILE" | awk '{print $1}')

echo "POC RETURN CODE: $POC_RC"
echo "Pre-run lines : $PRE_LINES"
echo "Post-run lines: $POST_LINES"
echo "Pre-run SHA   : $PRE_SHA"
echo "Post-run SHA  : $POST_SHA"

if [ "$PRE_SHA" = "$POST_SHA" ] && [ "$PRE_LINES" = "$POST_LINES" ]; then
  echo "SOURCE IMMUTABILITY PASS"
else
  echo "SOURCE IMMUTABILITY FAIL"
fi
```

***

## Observed Output

Relevant output from the final successful browser-to-chain run:

```
Running 1 test using 1 worker

[portal-e2e] › projects/portal/tests/portal-stale-registration-name-final.spec.ts:398:5 › AUDIT PoC - Portal stale registration actor › shows Bravo while stale Alpha actor submits register(Alpha)

[AUDIT PRE] Alpha=pxch.eth; Bravo=portal-bravo-mtwxqxch.eth; bothAvailable=true; wallet=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

[AUDIT PRICE A] current=pxch.eth; displayed-total-usd=160.11

[AUDIT COOLDOWN] Ready in 60s

[AUDIT PRE-SWITCH] Alpha=pxch.eth; registrar-register-requests=0; AlphaAvailable=true

[AUDIT BLOCKER] Your registration is in progress. Leaving may interrupt it and you could lose your commitment. Are you sure you want to leave?

[AUDIT SWITCH] current=portal-bravo-mtwxqxch.eth; Bravo-total-usd=8.01; abandoned-Alpha-total-usd=160.11

[AUDIT STALE REQUEST] current-ui=portal-bravo-mtwxqxch.eth; current-total-usd=8.01; pending-register-label=pxch; wallet-send-pending=1

[AUDIT POST] Alpha=pxch.eth available=false; Bravo=portal-bravo-mtwxqxch.eth available=true; current-Bravo-total-usd=8.01; actual-USDC-debit=160.109598

============================================================
CRITICAL PORTAL STALE REGISTRATION CONFIRMED
============================================================
Abandoned Alpha: pxch.eth
Current Portal name: portal-bravo-mtwxqxch.eth
Current Bravo displayed total USD: 8.01
Pending/approved registrar label: pxch
Alpha registered: true
Bravo still available: true
Actual USDC debit: 160.109598
============================================================

1 passed (1.3m)

POC RETURN CODE: 0

Pre-run lines : 966
Post-run lines: 966

Pre-run SHA   : 0c5ded13015f25fe6b8fb7634e9c9f1622efc584168a4a342cfa7b6e9b6bbcd1
Post-run SHA  : 0c5ded13015f25fe6b8fb7634e9c9f1622efc584168a4a342cfa7b6e9b6bbcd1

SOURCE IMMUTABILITY PASS
```

***

## What the PoC Proves

1. Alpha and Bravo are both available before the vulnerable sequence.
2. Portal displays Alpha at `$160.11`.
3. Alpha progresses through the real registration flow until `commitmentCooldown`.
4. **Before the name switch there are zero final registrar `register()` wallet requests**: `registrar-register-requests=0`.
5. Alpha remains available before the switch.
6. The user closes the transaction modal.
7. The user selects Bravo through Portal's own SPA search.
8. Portal presents the real warning that leaving may interrupt the registration and lose the commitment.
9. The user accepts that warning.
10. Portal successfully becomes Bravo and displays Bravo's `$8.01` checkout.
11. The abandoned Alpha registration actor remains alive.
12. When Alpha's real commitment cooldown expires, the actor creates a **new post-switch registrar transaction**.
13. Immediately before authorization, the current application resource is Bravo.
14. At that same boundary, a real wallet `sendTransaction` request is pending.
15. The PoC decodes that actual registrar calldata and obtains label `pxch`, i.e. Alpha.
16. Therefore:

```
current Portal resource = Bravo
pending registrar label = Alpha
```

17. The user authorizes the wallet request.
18. Alpha becomes unavailable/registered on the authoritative local registrar.
19. Bravo remains available.
20. The victim's ERC-20 balance decreases by `160.109598 USDC`.
21. The `$160.11` Alpha vs `$8.01` Bravo amount difference is supporting material-impact evidence; the Critical claim itself is the post-switch wrong-name transaction.
22. The exact 966-line PoC remained byte-for-byte unchanged during execution.
23. The run completed with `1 passed (1.3m)`, `POC RETURN CODE: 0`, and `SOURCE IMMUTABILITY PASS`.

The reproduced mismatch is:

```
BEFORE SWITCH
=============
final registrar requests = 0
Alpha available          = true

CURRENT APPLICATION REVIEW
==========================
Name = Bravo

APPLICATION-GENERATED WALLET REQUEST
====================================
ETHRegistrar.register("pxch", ...)
                      ^ Alpha

AUTHORITATIVE RESULT
====================
Alpha registered
Bravo still available
```

Material consequence observed in the same run:

```
abandoned Alpha checkout = $160.11
current Bravo checkout   = $8.01
actual stale Alpha debit = 160.109598 USDC
```

This demonstrates the scoped transaction-integrity impact: **after Portal has transitioned to Bravo, it generates and successfully executes a registrar transaction for the abandoned Alpha registration.**
