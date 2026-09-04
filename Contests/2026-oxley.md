# Oxley
Oxley Private Audit || Core and Roundtable V2 — launch hooks, V4 pools, seats, and volume rewards on Robinhood Chain || Robinhood Chain (testnet 46630)

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-attacker-holding-an-active-seat-will-permanently-steal)|An attacker holding an active seat will permanently steal the entire roundtable reward entitlement of any LP that activates or rolls, as the R2.1 block.number membership clock treats ~13 seconds of Robinhood L2 swaps as a single block|HIGH|
|[H-02](#h-02-attacker-holding-a-cheap-lane-seat-will-steal)|An attacker holding a cheap-lane seat will steal another seat’s volume generated roundtable rewards as allocate pays tick distance instead of quote volume|HIGH|
|[H-03](#h-03-traders-will-be-unable-to-buy-through-official-settle-after)|Traders will be unable to buy through official settle-after V4 routers on freshly seeded V1/V4 pools as the hook takes real USDG from PoolManager before the caller settles|HIGH|
||||
|[M-01](#m-01-inverted-launches-will-initialize-one-tick-off)|Inverted launches will initialize one tick off the reciprocal price as orientedInitialTick uses -tick instead of -tick-1|MEDIUM|
|[M-02](#m-02-attackers-will-grief-v1-v4-permit-launches)|Attackers will grief V1/V4 permit launches by front-running the victim’s EIP-2612 signature so createCoinWithPermit* reverts on a consumed nonce|MEDIUM|
|[M-03](#m-03-a-seat-opener-will-have-openseat-revert)|A seat opener will have openSeat revert on the quoted max as quoteSeat uses the tick-boundary sqrt instead of the live pool price|MEDIUM|
||||
|[L-01](#l-01-last-roundtable-seats-to-leave-a-lane)|Last Roundtable seats to leave a lane will permanently lock the per-lane reward remainder in the accumulator reserve|LOW|

----

## [H-01] An attacker holding an active seat will permanently steal the entire roundtable reward entitlement of any LP that activates or rolls, as the R2.1 block.number membership clock treats ~13 seconds of Robinhood L2 swaps as a single block

### Summary

R2.1 replaced the L2 `ArbSys.arbBlockNumber()` membership clock with Solidity `block.number` after concluding the precompile was missing. On Robinhood Chain (an Orbit chain) that global is the `L1` block estimate, so the `childBlock + 1` pending gate that the tests exercise with `vm.roll` as "one block" is really "one L1 tick"  measured live at `85–130` L2 blocks and up to 16 seconds. Because `rollSeat` re-enters `pendingCount` while it simultaneously installs the seat's liquidity in the destination band, a rolling LP serves real swap flow while `_allocateLane` treats it as absent: its share goes to whichever seats are already active, or entirely to the platform when the lane is otherwise empty. An attacker who pre-positions the only other active seat in the destination lane and back-runs the victim's mempool-visible `rollSeat` captures 100% of that flow, and the `~13s` window makes the back-run trivially reliable where a correct `L2 clock` would leave ~0.1s.

### Root Cause

The R2.1 clock reads the Solidity global directly, at OxleyRoundtableManagerR2.sol[ L961–L965](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L961-L965):

```solidity
function _childBlockNumber() private view returns (uint64 childBlock) {
    uint256 value = block.number;
    if (value > type(uint64).max) revert ChildClockOverflow(value);
    childBlock = value.toUint64();
}
```

On Robinhood Chain this is the L1 estimate, not the L2 height. Confirmed live on 2026-08-19 against `https://rpc.testnet.chain.robinhood.com (chainId 0xb626 = 46630)`: the raw NUMBER opcode, executed via a `state-override eth_call` of runtime `bytecode 0x4360005260206000f3`, returns `0xafcee1 = 11521761`, which is exactly the block's `l1BlockNumber` field  while the L2 height at that instant was `103803627` and `ArbSys(0x64).arbBlockNumber()` returned `0x62feafb = 103803643`. The R2.1 premise that the precompile is unimplemented and reverts `ChildClockInvalid` is false on the current testnet; the previous ArbSys implementation was the correct L2 clock.

Membership is then gated on that clock in three places that together produce the bug.

Activation records pending membership effective at `childBlock + 1` and snapshots a baseline, at OxleyRoundtableManagerR2.sol[ L345–L356](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L345-L356):

```solidity
uint64 effectiveBlock = childBlock + 1;
if (lane.pendingEffectiveBlock == 0) lane.pendingEffectiveBlock = effectiveBlock;
++lane.pendingCount;
endOfBlockIndex[poolId][seat.band][childBlock] = lane.rewardPerSeatIndex;
seat.eligibleBlock = effectiveBlock;
seat.rolloverBlock = childBlock;
seat.rewardDebt = INDEX_UNSET;
```

rollSeat repeats exactly the same pending re-entry, at OxleyRoundtableManagerR2.sol[ L567–L581](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L567-L581) and it does so after `unlockCallback`[ L610–L613](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L610-L613) has already added the seat's liquidity to the destination band. This is what makes the finding recurring rather than a one-time activation cost: the seat is a live LP absorbing swap flow in the new band while counted as pending.

Allocation ignores pending seats entirely, at OxleyRoundtableManagerR2.sol[ L772–L779](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L772-L779):

```solidity
_syncLane(lane, childBlock);
if (lane.activeCount == 0) return (0, laneBudget);   // whole lane budget -> platform
uint256 available = laneBudget + lane.rewardRemainder;
uint256 perSeat = available / lane.activeCount;      // pending seats excluded from the divisor
lane.rewardPerSeatIndex += perSeat;
endOfBlockIndex[poolId][laneIndex][childBlock] = lane.rewardPerSeatIndex;
```

`_syncLane` only flushes pending into active once the clock strictly advances, at OxleyRoundtableManagerR2.sol[ L818–L824](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L818-L824). Because `effectiveBlock = childBlock + 1`, the flush cannot happen until the L1 number ticks  i.e. after every L2 block in the current ~13s window.

The loss is made permanent by the baseline re-read at OxleyRoundtableManagerR2.sol[ L792–L795](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L792-L795):
```solidity
uint256 baseline = seat.rewardDebt;
if (baseline == INDEX_UNSET) {
    baseline = endOfBlockIndex[poolId][seat.band][seat.rolloverBlock];
}
amount = lane.rewardPerSeatIndex - baseline;
```

The seat's baseline key is `rolloverBlock == childBlock`, and every in-window allocation overwrites `endOfBlockIndex[...][childBlock]` with the newest index (L779). So when the seat finally settles, its baseline equals the index at the end of the window, and amount excludes the entire window. Nothing is ever back-paid.

When the lane budget is skipped, redirected is added straight to the platform payout at OxleyRoundtableManagerR2.sol[ L431–L433](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L431-L433) (and [L401–L402](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L401-L402) for the stationary path), so a sole-seat lane in blackout hands the platform 100% of that flow irreversibly.

Finally, repositioning is gated on the same broken clock at OxleyRoundtableManagerR2.sol[ L537](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L537):
```solidity
if (childBlock < seat.eligibleBlock) revert PendingRollover(seat.eligibleBlock);
```

so a seat that has just rolled is locked out of rolling again for the remainder of the `~13s` window, even if price runs away through further bands  precisely during the volatile bursts when rolling matters and when reward flow is highest.

**What the code should do** is read the L2 height, `ArbSys(0x64).arbBlockNumber()`, which the live probe above shows is implemented and returns the L2 number. With that clock the exclusion window is one `~0.1s L2 block`, matching design intent and the `vm.roll` semantics the test suite encodes.

### Internal Pre-conditions

- A canonical R2 pool is seeded and swaps accrue a roundtable budget  measured at 5 bps of gross `quote` (50_000 base units on a 100e6 buy), or 5 bps plus the creator half under Roundtable Boost. This is the value stream the finding drains.
- A seat is in `pendingCount` on a lane because `activateSeat` or `rollSeat` succeeded in the current `block.number` window, so `eligibleBlock == block.number + 1`. The rollSeat path is the important one: it recurs for the life of the seat and leaves live liquidity in the destination band.
- Further L2 swaps occur before `block.number` increments. On Orbit that window is the rest of the L1 block, so it is 85–130 L2 blocks wide rather than zero.
- For the maximal (100% capture) case, exactly one other seat is active in the destination lane,  the attacker's. For the platform-diversion case, no other seat is active, and `_allocateLane` redirects the whole lane budget.

### Attack Path

This has an honest-user path and an attacker-directed path. Both are proven.

**Honest-user path (happens continuously, no attacker):**
- An LP holds an active seat in `lane k` and price moves into `lane k+1's` entry window.
- The LP calls `rollSeat. unlockCallback` removes liquidity from band k and installs it in `band k+1` in the same unlock, so the seat is immediately a live LP there.
- `rollSeat` then sets `toLane.pendingCount += 1` and `seat.eligibleBlock = block.number + 1 (L567–L581)`.
- Every swap for the next 85–130 L2 blocks calls `allocate` with the same `L1 childBlock. _syncLane` does not flush, so the rolled seat is excluded from `activeCount`.
- Its share is paid to already-active seats, or — if the lane has no other active seat — the entire lane budget is added to platformAmount and leaves the roundtable permanently.
- When the L1 number finally ticks and the seat settles, its baseline has been overwritten to the end-of-window index, so the window's flow is never back-paid.

**Attacker-directed path (maximises the loss and redirects it to the attacker):**
- The attacker opens and activates a seat in `lane k+1` and lets it `sync` to `activeCount == 1`. Cost is the ordinary seat deposit, which remains fully theirs.
- The attacker watches the mempool for any victim's `rollSeat` into `lane k+1` (public, unencrypted).
- The victim's roll lands; the victim is now `pendingCount == 1` with live liquidity in `lane k+1`, and the attacker is the only `activeCount`.
- The attacker back-runs with a swap in the same `L1` window. `_allocateLane` computes `perSeat = laneBudget / 1` and assigns the whole budget to the attacker.
- The attacker collects `100%` of the lane budget for flow that the victim's liquidity also served. The victim collects `0` and can never recover it.
- The victim additionally cannot `rollSeat` again for the rest of the window (L537), so it cannot escape the lane even if price leaves it.

### Impact

Roundtable rewards are `5` bps of gross USDG volume (30 bps of the swap fee under Boost: the creator half plus 5 bps). This causes `100%` of a pending seat's entitlement for the blackout window to be permanently reassigned  to an attacker's active seat, to other honest active seats, or to the platform. It is never back-paid, and it recurs on every activation and every roll.

### poc

The decisive Solidity test, from packages/contracts/test/R2AuditRollBlackout.t.sol:
```solidity
/// @dev Directed theft: the victim's rollSeat is public in the mempool. An attacker
///      holding the only other active seat in the destination lane back-runs it with a
///      swap inside the same L1 tick and captures 100% of the lane budget the victim's
///      live liquidity actually served. The ~13s tick makes this trivial to land; with a
///      correct L2 clock the window would be a single ~0.1s block.
function testAttackerBackrunsVictimRollToStealLaneRewards() public {
    (address token, bytes32 poolId) =
        _launch("Backrun Roll", "BKRNRL", keccak256("backrun-roll"), OxleyCoinFactoryR2.IncentiveMode.None);
    vm.warp(block.timestamp + 30 minutes);
    _moveToNormalizedTick(token, BAND0_MID);

    address attacker = secondTrader;
    vm.prank(attacker);
    IERC20(token).approve(address(roundtable), type(uint256).max);
    vm.prank(attacker);
    usdg.approve(address(roundtable), type(uint256).max);

    // Victim takes a seat in lane 0 and becomes fully active.
    uint256 victim = _openExactAs(trader, token, 0);
    vm.warp(block.timestamp + 90 seconds);
    roundtable.activateSeat(victim);
    _advanceChildBlock();

    // Attacker pre-positions an active seat in lane 1, the victim's destination.
    _moveToNormalizedTick(token, BAND1_MID);
    uint256 attackerSeat = _openExactAs(attacker, token, 1);
    vm.warp(block.timestamp + 90 seconds);
    roundtable.activateSeat(attackerSeat);
    _advanceChildBlock();

    // Victim rolls to follow price. Drain its lane-0 earnings so the measurement below
    // isolates the blackout.
    _rollTo(victim, 1);
    vm.prank(trader);
    roundtable.collectRewards(victim);

    (uint64 activeCount, uint64 pendingCount,,,) = roundtable.laneState(poolId, 1);
    assertEq(activeCount, 1, "only the attacker counts as active");
    assertEq(pendingCount, 1, "victim is pending despite live liquidity");

    // Attacker's back-run, same block.number tick as the victim's roll.
    (uint256 budget, uint256 reserved,) = _buyAndReadAllocation(token, 100e6);
    assertGt(budget, 0, "back-run produced budget");
    assertEq(reserved, budget, "full lane budget reserved for the single active seat");

    vm.prank(attacker);
    uint256 attackerTook = roundtable.collectRewards(attackerSeat);
    vm.prank(trader);
    uint256 victimTook = roundtable.collectRewards(victim);

    assertEq(attackerTook, budget, "attacker captures 100% of the lane budget");
    assertEq(victimTook, 0, "victim is paid nothing for serving the same swap");

    // Without the blackout the two seats would have split it evenly.
    _advanceChildBlock();
    (uint256 fairBudget,,) = _buyAndReadAllocation(token, 100e6);
    vm.prank(trader);
    assertEq(roundtable.collectRewards(victim), fairBudget / 2, "fair share is half");
    emit log_named_uint("stolen base units per back-run", attackerTook - fairBudget / 2);
}
```

Supporting test proving the platform-diversion variant and that live liquidity is unpaid:
```solidity
function testRollReEntersBlackoutWhileLiquidityIsLive() public {
    (address token, bytes32 poolId) =
        _launch("Roll Blackout", "RLLBLK", keccak256("roll-blackout"), OxleyCoinFactoryR2.IncentiveMode.None);
    vm.warp(block.timestamp + 30 minutes);
    _moveToNormalizedTick(token, BAND0_MID);

    uint256 seatId = _openExact(token, 0);
    vm.warp(block.timestamp + 90 seconds);
    roundtable.activateSeat(seatId);
    _advanceChildBlock();

    // Seat is fully active in lane 0 and earns normally.
    _buyExactInput(token, 100e6, _unboundedLimit(_buyZeroForOne(factory.poolKeyFor(token))), trader);
    vm.prank(trader);
    uint256 earnedWhileActive = roundtable.collectRewards(seatId);
    assertGt(earnedWhileActive, 0, "active seat must earn");

    // Price moves into band 1's entry window; the seat rolls to follow it.
    _moveToNormalizedTick(token, BAND1_MID);
    _advanceChildBlock();
    _rollTo(seatId, 1);

    // Drain everything legitimately earned in lane 0 (the roll settles it), so the
    // balance below measures only what happens during the post-roll blackout.
    vm.prank(trader);
    roundtable.collectRewards(seatId);

    // Liquidity is live in lane 1 immediately, but the seat sits in pendingCount.
    (uint64 activeCount, uint64 pendingCount,,,) = roundtable.laneState(poolId, 1);
    assertEq(activeCount, 0, "no active seats in destination lane");
    assertEq(pendingCount, 1, "rolled seat is pending");
    (,,,,,,, uint128 liquidity,,,) = roundtable.seats(seatId);
    assertGt(liquidity, 0, "seat liquidity is live in the new band");

    // Swaps inside the same block.number tick: seat is the ONLY LP in lane 1.
    uint256 reserveBefore = accumulator.roundtableReserve(poolId);
    (uint256 budget, uint256 reserved, uint256 platform) = _buyAndReadAllocation(token, 100e6);
    assertGt(budget, 0, "swap produced roundtable budget");

    // 100% of the lane budget is permanently redirected to the platform.
    assertEq(reserved, 0, "nothing reserved for the roundtable");
    assertEq(platform, budget, "entire lane budget diverted to platform");
    assertEq(accumulator.roundtableReserve(poolId) - reserveBefore, 0, "roundtable reserve untouched");

    vm.prank(trader);
    assertEq(roundtable.collectRewards(seatId), 0, "rolled seat earns nothing for the whole tick");
}
```

### Recommended fix:
 restore the L2 clock, `ArbSys(0x64).arbBlockNumber() `, so the pending window is the single L2 block the design intends.

---

## [H-02] An attacker holding a cheap-lane seat will steal another seat’s volume generated roundtable rewards as allocate pays tick distance instead of quote volume

The hook mints the roundtable budget from `actualGrossQuote` (a volume quantity) and passes the `swap’s start/end sqrt` prices into `allocate`. `allocate` drops those sqrts and weights each lane by `segmentTicks / pathTicks`. Ticks are log-price, so the cheap lane served less `USDG` but is paid the same. The difference is taken from the high-volume seat and credited to the attacker, who then `collectRewards` it.

### Root Cause

The fee is volume-derived in [OxleyLaunchHookR2.sol L445–L446](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyLaunchHookR2.sol#L445-L446):

```solidity
uint256 roundtableBudget = FullMath.mulDiv(execution.accounting.actualGrossQuote, ROUND_TABLE_GROSS_BPS, 10_000);
```

The hook then hands the real path, including sqrts, to the manager at [L467–L476](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyLaunchHookR2.sol#L467-L476). AllocationPath carries startSqrtPriceX96 / endSqrtPriceX96 ([IOxleyRoundtableManagerR2.sol L43–L49](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/interfaces/IOxleyRoundtableManagerR2.sol#L43-L49)).

allocate at [OxleyRoundtableManagerR2.sol L368–L373](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L368-L373) keeps only the ticks. _nextLaneBudget at [L438–L446](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L438-L446) is:

```solidity
laneBudget = mulDiv(insideBudget, segmentHigh - segmentLow, insideDistance);
```

**Wrong formula:** `laneBudget_i = budget * ticks_i / ticks_total`.
Right formula, given a fee taken from quote volume: `laneBudget_i = budget * quoteVolume_i / quoteVolume_total (or L * |Δsqrt|_i`, which the unused sqrts already determine).


### Internal Pre-conditions

- Two (or more) seats are active in different lanes of one pool,  the victim has a claim on the volume-derived pot.

- A swap’s normalized path crosses those lanes with unequal quote volume per tick (the default: lower ticks have smaller |Δsqrt| per tick).

- Empty lanes in between only change how much is redirected to the platform; they do not restore the victim’s volume-fair share.

### External Pre-conditions

- None. `openSeat + activateSeat` in the cheaper lane is unprivileged and costs the same 2,500 USDG as the victim.

### Attack Path

1. Attacker opens and activates a seat in a low-price lane (band 0). The victim does the same in a higher lane (band 2). Same USDG debit.

2. Price is in the victim’s band. A sell (or buy) crosses back through both lanes. The 5 bps (or Boost) pot is minted from the total quote volume of that swap.

3. `allocate` splits that pot by tick width. Equal-width occupied segments get equal rewardPerSeatIndex bumps, even though most of the quote  and therefore most of the fee  was traded in the victim’s lane.

4. Attacker calls `collectRewards` and withdraws the surplus. The victim’s claimable is permanently short that amount. Repeat on every multi-lane swap.

Honest-user path is the same: any LP in a cheaper band is overpaid at the expense of LPs in richer bands. The attacker path is just choosing the cheaper band on purpose.

### Impact

Stealing, yes. The stolen object is the victim seat’s share of the volume-derived roundtable reserve. That reserve is real USDG (PoolManager 6909 claims) that `creditRoundtable` then `claimRoundtableFor` pays out. It is not a book-entry rounding remainder: the attacker’s wallet goes up by the exact amount the victim’s wallet does not.

- The loss is confined to the protocol incentive those seats are opened to earn,  the 5 bps stream (30 bps of swap fee under Roundtable Boost).

Measured (FOUNDRY_PROFILE=r2 forge test --match-test testTickWeightVsVolumeTheft -vv):

- Path: sell -370620 → -384540. Occupied lanes 0 and 2 are both 3,480 ticks.

- Quote volume: lane 0 4,451,666,845 (34.20%), lane 2 8,563,290,812 (65.80%).

- Tick share / paid: 3,484,308 each (50/50).

- Volume-fair: lane 0 2,383,561, lane 2 4,585,055.

- Stolen this swap: 1,100,747 USDG-6 from the victim to the attacker.

- That is 15.79% of the 6,968,616 occupied pot and 24% of the victim’s volume-fair claim (1100747 / 4585055).

- Claims matched the tick budgets exactly (attackerClaim 3484308, victimClaim 3484308).


### PoC

From packages/contracts:

```bash
FOUNDRY_PROFILE=r2 forge test --match-test testTickWeightVsVolumeTheft -vv
```

Passing 2026-08-21 in test/W4_A1_LaneBudgetGeometry.t.sol. Log lines: volShare0_bps 3420, tickShare0_bps 5000, attackerOver 1100747, misBps 1579.

```solidity
function testTickWeightVsVolumeTheft() public {
    (address token, bytes32 poolId) = _boot(true, "A1V", keccak256("a1v"));
    (uint256 attacker, uint256 victim) = _openPair(token, 0, 2);
    vm.prank(secondTrader);
    roundtable.collectRewards(attacker);
    vm.prank(trader);
    roundtable.collectRewards(victim);
    _measureTheft(token, poolId, attacker, victim);
}
```

Fix: weight `_nextLaneBudget` by `|Δsqrt| (or by quote delta)` over each lane using `path.startSqrtPriceX96 / path.endSqrtPriceX96` already in the calldata, not by segmentHigh - segmentLow.

---

## [H-03] Traders will be unable to buy through official settle-after V4 routers on freshly seeded V1/V4 pools as the hook takes real USDG from PoolManager before the caller settles

### Summary

V1 and V4 launch hooks settle the `0.5%` Oxley fee with `poolManager.take(quote, …)` inside `afterSwap`, i.e. before a `settle-after` router pays the buyer's input. Official v4-periphery `V4Router / Universal` Router pay after `poolManager.swap` returns. Seeding is coin-only, so a fresh `PoolManager` holds `0` quote and every such buy reverts `ERC20InsufficientBalance(PoolManager, 0, firstTake)`. The revert recurs whenever `fee > PoolManager.balanceOf(quote)` (for example after deep sell-offs). In repo routers `prepay`, so the existing suite never sees it.

### Root Cause

- OxleyV4LaunchHook.sol[ L330](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyV4LaunchHook.sol#L330) calls `_settleFee` during `afterSwap`. The fee is pulled as real ERC20:

OxleyV4LaunchHook.sol[ L426–L444](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyV4LaunchHook.sol#L426-L444)

```solidity
function _settleFee(...) private {
    (uint256 creatorShare, uint256 platformShare) = OxleyFeeMath.split(totalFee);
    Currency quote = Currency.wrap(MOCK_USDG);
    if (totalFee != 0) {
        if (creator == FEE_RECIPIENT) {
            poolManager.take(quote, creator, totalFee);
        } else {
            if (creatorShare != 0) poolManager.take(quote, creator, creatorShare);
            if (platformShare != 0) poolManager.take(quote, FEE_RECIPIENT, platformShare);
        }
    }
}
```

The same take lives on the V4 deployable hook:

OxleyLaunchHookV4.sol[ L561–L576](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyLaunchHookV4.sol#L561-L576)

`take` transfers the `PoolManager's` current ERC20 balance. It does not consume the swapper's still-unsettled debit. Seed `(OxleyV4LaunchHook.sol L262–L273 / V4 equivalent)` requires `quoteDelta == 0`, so a new pool contributes `0 USDG`. The hook should keep the fee as a hook `claim / BeforeSwapDelta` and let recipients pull after the `unlock`, or reject non-prepaid senders. R2 already does the former (OxleyLaunchHookR2.sol L502–L511 mints to the accumulator).

This is not the known frozen-recipient issue. That known item is “take to a Paxos-frozen creator/platform reverts the swap.” This finding is settlement ordering: take against a zero (or too-small) `PoolManager quote` balance.

### Internal Pre-conditions

- A V1 (OxleyV4LaunchHook) or V4 (OxleyLaunchHookV4) pool is seeded. Seed settles coin only, so `PoolManager` quote balance starts at 0 (or later falls below the next fee).
- A buyer routes through a settle-after unlocker (swap first, then `sync/transferFrom/settle`). This is the official `V4Router / Universal Router` pattern, not a broken custom router.
- `totalFee > 0 (any buy at or above the 200-wei fee floor)`.

### scenario
- Creator launches a coin. Arc/hook seed deposits the full coin supply; `USDG` in `PoolManager` remains 0.
- A trader buys via `V4Router / Universal Router / any settle-after` unlocker.
- `beforeSwap` reserves the `fee` as `BeforeSwapDelta`. The pool swap runs.
- `afterSwap` → `_settleFee` /` _settleSwapFee` → `poolManager.take(quote, creator, creatorShare)`.
- `PoolManager's` USDG balance is 0, so the ERC20 transfer reverts `ERC20InsufficientBalance(PoolManager, 0, creatorShare)`. The whole unlock reverts. Trader balances are unchanged.
- A prepaid control on the same pool succeeds. After that prepaid buy leaves quote in `PoolManager`, a later settle-after buy also succeeds. The failure is therefore `settlement` order, not “buys are generally broken.”
- After later sells drain `PoolManager` quote below the next `fee`, the same revert returns.

No privileged role is required. Honest users on the most common V4 integration path are the victims.

### Impact

- Buy path DoS for settle-after routers on every freshly seeded V1/V4 pool, and again whenever global PoolManager quote < fee. No funds are stolen; the revert is atomic.

- In-repo routers (OxleyProofRouter, CheckpointTwoRouter, the R2 adapter) all prepay, which is why the existing suite is green. The hook does not require those routers: `_validateSwapPool` only checks registration.

### poc

```solidity
function testV1SettleAfterBuyRevertsOnFreshPoolAtTake() public {
    // launch V1 pool; seed is coin-only
    assertEq(usdg.balanceOf(address(manager)), 0);
    SettleAfterRouter settleAfter = new SettleAfterRouter(manager);
    vm.prank(trader);
    usdg.approve(address(settleAfter), type(uint256).max);

    uint256 traderBefore = usdg.balanceOf(trader);
    uint256 totalFee = OxleyFeeMath.feeFromGross(100e6);
    (uint256 firstTake,) = OxleyFeeMath.split(totalFee); // V1 is 80/20

    vm.prank(trader);
    try settleAfter.buyExactInput(key, zeroForOne, 100e6, unboundedLimit) {
        revert("settle-after buy should revert");
    } catch (bytes memory reason) {
        // inner take: ERC20InsufficientBalance(PoolManager, 0, creatorShare)
        assertTrue(_contains(reason, IERC20Errors.ERC20InsufficientBalance.selector));
        assertTrue(_contains(reason, abi.encode(address(manager), uint256(0), firstTake)));
    }
    assertEq(usdg.balanceOf(trader), traderBefore);
    assertEq(usdg.balanceOf(address(manager)), 0);
}

SettleAfterRouter.unlockCallback calls manager.swap first, then settles deltas — the official periphery order. Full routers and the V4 twin are in packages/contracts/test/AuditPocSettleAfter.t.sol.
```

---

## [M-01] Inverted launches will initialize one tick off the reciprocal price as orientedInitialTick uses -tick instead of -tick-1

### Summary

Uniswap tick inversion for a current price is `-tick - 1` because ticks are lower bounds of price intervals `(sqrtP(t) * sqrtP(-t-1) = 2^96)`. Oxley initializes inverted pools at `-CANONICAL_INITIAL_TICK (391500)` and then normalizes with `-actual - 1`, so the launch tick becomes `-391501`, which is strictly below `FIRST_REWARD_BOUNDARY (-391500)`. Stationary swaps at that tick send the entire roundtable budget to the platform.

### Root Cause

[LaunchProfileV2.sol L156–L161](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/LaunchProfileV2.sol#L156-L161) sets the inverted initial tick to -CANONICAL_INITIAL_TICK (391500), not 391499.

[OxleyLaunchProfileR2.sol L29–L31](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyLaunchProfileR2.sol#L29-L31) then normalizes inverted ticks as -actualTick - 1.

Canonical launch: actual -391500 → normalized -391500 (inside envelope [FIRST_REWARD_BOUNDARY, LAST_REWARD_BOUNDARY)).

Inverted launch: actual 391500 → normalized -391501. [_insideRewardEnvelope](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L967-L969) requires tick >= -391500, so -391501 fails and [_allocateStationary](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L397-L400) assigns platformAmount = budget.

The gas fixture already uses the correct reciprocal ticks 391499 / 245339 in [R2AtomicGas.t.sol L121–L122](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/test/R2AtomicGas.t.sol#L121-L122), which is the x ≡ y pair the initializer violates.

### Internal Pre-conditions

- Token address > USDG so the coin is currency1.

- A swap that does not cross a tick while the pool is still at the initial price (common for tiny first buys).

### External Pre-conditions

- None beyond a normal inverted launch.

### Attack Path

1. Launch a token whose CREATE2 address is greater than USDG (factory salts already target both orientations in tests).

2. Before price leaves the initial tick, swap (stationary path). Roundtable allocate sees normalized start = end = -391501, outside the envelope, and redirects the 5 bps budget to platformSwapClaimable.

3. A 1-tick buy that lands on -391500 does the same: moving path insideLow = -391500, insideHigh = -391500, so insideHigh <= insideLow and the whole budget still goes to the platform.

4. After the pool moves further into lane 0, later swaps behave. The first-tick slice is permanently platform-owned. Fee-policy tests miss this because they move to the band midpoint (-384540) first.

### Impact

One-tick price offset (~0.01%). First stationary inverted swaps divert 5 bps of those notionals from roundtable LPs to the platform. 
### PoC

How to run from packages/contracts:

```bash
FOUNDRY_PROFILE=r2 forge test --match-test testInvertedInitialTickIsOutsideRewardEnvelope -vv
```

Measured results (passing 2026-08-17):

```text
[PASS] testInvertedInitialTickIsOutsideRewardEnvelope().
```

Inverted pool initial tick is 391500, normalized -391501, which is < FIRST_REWARD_BOUNDARY (-391500).

```solidity
function testInvertedInitialTickIsOutsideRewardEnvelope() public {
    bytes32 salt = _saltForTokenOrientation(false, "Inv Tick", "INVTCK", keccak256("inv-tick"));
    (address token,) = _launch("Inv Tick", "INVTCK", salt, OxleyCoinFactoryR2.IncentiveMode.None);
    PoolKey memory key = factory.poolKeyFor(token);
    assertTrue(Currency.unwrap(key.currency0) != token);

    (, int24 actualTick,,) = manager.getSlot0(key.toId());
    assertEq(actualTick, 391_500);
    int24 normalized = launchProfile.normalizeEligibilityTick(actualTick, false);
    assertEq(normalized, -391_501);
    assertTrue(normalized < int24(-391_500)); // FIRST_REWARD_BOUNDARY
}
```

---

---

## [M-02] Attackers will grief V1/V4 permit launches by front-running the victim’s EIP-2612 signature so createCoinWithPermit* reverts on a consumed nonce

### Summary

V1 and V4 factories call USDG.permit unconditionally whenever `permitData.value != 0`. An attacker can submit the same signature to `USDG.permit` first, burning the nonce. The victim’s `createCoinWithPermit`* then reverts. The front-run leaves `allowance == CREATION_FEE`, so the victim recovers in the next transaction via `createCoin / createCoinFor`. R2’s skip-if-exact allowance check is the fix.

### Root Cause

V1 always consumes the signature:

[OxleyCoinFactory.sol L229–L241](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyCoinFactory.sol#L229-L241)

```solidity
if (permitData.value != 0) {
    IERC20Permit(address(MOCK_USDG))
        .permit(msg.sender, address(this), CREATION_FEE_MOCK_USDG, permitData.deadline, permitData.v, permitData.r, permitData.s);
}
MOCK_USDG.safeTransferFrom(msg.sender, FEE_RECIPIENT, CREATION_FEE_MOCK_USDG);
```

V4 is the same pattern:

[OxleyCoinFactoryV4.sol L254–L266](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyCoinFactoryV4.sol#L254-L266)

EIP-2612 nonces are single-use. A public mempool copy of (v,r,s) is a valid permit call for anyone. After the nonce increments, the factory’s second permit recovers a digest for the new nonce and reverts.

R2 already implements the recovery-safe pattern:

[OxleyCoinFactoryR2.sol L459–L473](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyCoinFactoryR2.sol#L459-L473)

```solidity
if (USDG.allowance(msg.sender, address(this)) != CREATION_FEE_USDG) {
    IERC20Permit(address(USDG)).permit(...);
}
```

What the V1/V4 code should do is skip permit when allowance is already exact (R2) or treat ERC2612InvalidSigner / used-nonce as success when allowance matches.


### Internal Pre-conditions

- Victim uses createCoinWithPermit or createCoinWithPermitFor on V1 or V4 (not R2).

- The signed permit is the canonical creation-fee permit (value == CREATION_FEE, spender = factory, owner = msg.sender).

### External Pre-conditions

- The signature is visible in the mempool (public L2). No privileged role.

### Attack Path

1. Victim signs Permit(owner=victim, spender=factory, value=1e6, nonce=N, deadline) and submits createCoinWithPermit*.

2. Attacker copies (v,r,s) and calls USDG.permit(victim, factory, 1e6, deadline, v, r, s). Nonce becomes N+1. Allowance is exactly 1_000_000.

3. Victim’s factory call executes permit again. The digest now binds nonce N+1. Signature verification reverts. Ticker/salt are not consumed (the revert is before those writes finish; the whole nonReentrant tx rolls back).

4. Victim calls createCoin / createCoinFor with the allowance the attacker just set. Launch succeeds. Attacker profit is 0; victim lost one failed-tx gas.

### Impact

Gas-only grief of the permit launch entry point on V1 and V4. No USDG stolen, no ticker stolen, no salt consumed. 


### PoC

How to run from packages/contracts:

```bash
FOUNDRY_PROFILE=r2 forge test --match-contract AuditPocPermitGriefTest -vv
```

Measured results (passing 2026-08-19): 2/2.

- testV1PermitFrontRunGriefsThenNonPermitRecovers

- testV4PermitFrontRunGriefsThenNonPermitRecovers

In both: after the attacker’s permit, nonces(owner) == N+1 and allowance == 1_000_000; createCoinWithPermit* reverts; createCoin / createCoinFor deploys the coin and consumes the allowance.

```solidity
function testV1PermitFrontRunGriefsThenNonPermitRecovers() public {
    OxleyCoinFactory.PermitData memory permitData = _signV1(factory, usdg.nonces(creator));
    uint256 nonceBefore = usdg.nonces(creator);

    vm.prank(attacker);
    IERC20Permit(address(usdg)).permit(
        creator, address(factory), permitData.value, permitData.deadline, permitData.v, permitData.r, permitData.s
    );
    assertEq(usdg.nonces(creator), nonceBefore + 1);
    assertEq(usdg.allowance(creator, address(factory)), 1_000_000);

    vm.prank(creator);
    vm.expectRevert();
    factory.createCoinWithPermit("Griefed", "GRF1", keccak256("v1-grief"), permitData);

    vm.prank(creator);
    (address coin,) = factory.createCoin("Recovered", "REC1", keccak256("v1-recover"));
    assertTrue(coin.code.length > 0);
    assertEq(usdg.allowance(creator, address(factory)), 0);
}
```

---

## [M-03] A seat opener will have openSeat revert on the quoted max as quoteSeat uses the tick-boundary sqrt instead of the live pool price

`quoteSeat` builds the debit from `TickMath.getSqrtPriceAtTick(actualTick)`, which is the lower edge of the current tick. `modifyLiquidity` inside the unlock charges from `slot0.sqrtPriceX96`, which sits strictly above that edge whenever the pool is intra-tick. The quoted `maxTokenAmount/maxUsdgAmount` is therefore too small, and `openSeat` reverts `InsufficientMaximum`. An unprivileged swap that stops inside the tick is enough to put any later opener on this path.

### Root Cause

[OxleyRoundtableManagerR2.sol L638–L643](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L638-L643) reads only the tick, then [L901–L918](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L901-L918) reconstructs price as the tick boundary:

```solidity
actualTick = _currentActualTick(key); // slot0.tick only
(uint256 amount0, uint256 amount1) = _positionAmountsForBand(band, actualTick, coinIsCurrency0, liquidity);

uint160 sqrtCurrent = TickMath.getSqrtPriceAtTick(actualTick);
```

`_currentActualTick` at [L888–L891](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L888-L891) discards sqrtPriceX96. v4-core’s mint uses the live sqrt. Wrong formula: amount = f(sqrt(tick)). Right formula: `amount = f(slot0.sqrtPriceX96)`.

### Internal Pre-conditions

- A canonical R2 pool is seeded and a band’s entry window is reachable.

- Current price is intra-tick: getSqrtPriceAtTick(tick) < slot0.sqrtPriceX96 < getSqrtPriceAtTick(tick+1).

- The opener passes quoteSeat’s exact tokenAmount/usdgAmount as the max (the integration the view is written for).

### External Pre-conditions

- Any prior swap may leave the pool intra-tick. No privileged role.

### Attack Path

1. Victim (or a helper) moves price to a band midpoint, then a swap nudges sqrtPrice toward the next tick without crossing.

2. Victim calls quoteSeat and submits openSeat with those exact maxima.

3. Unlock `modifyLiquidity` charges the live-sqrt amounts. One leg exceeds the quoted max.

4. `openSeat` reverts InsufficientMaximum(required, quoted). Retry with slack succeeds; nothing is taken.

### Impact

Grief / broken quote, not theft. Measured on the passing PoC:

- Canonical band 0: quoted USDG 2_500_000_000, actual 2_500_425_321, revert InsufficientMaximum(2500425321, 2500000000) — +425,321 units (0.425 USDG, ~1.7 bps).

- Inverted band 0: quoted token under-stated by 21_292_144_011_178_833_508_433 wei (~1.7 bps); USDG quote was slightly high.

### PoC

How to run from packages/contracts:

```bash
FOUNDRY_PROFILE=r2 forge test --match-test testQuoteUnderstatesUsdgIntraTickCanonical -vv
```

Measured (2026-08-21): [PASS] testQuoteUnderstatesUsdgIntraTickCanonical and testQuoteUnderstatesTokenIntraTickInverted in test/W4_A2_SeatLiquidity.t.sol.

```solidity
function testQuoteUnderstatesUsdgIntraTickCanonical() public {
    address token = _prepIntraTick(true, "A2 Qte Can", "A2QC", keccak256("a2-qte-can"));
    _fillShot(token);
    assertFalse(shot.tightOk, "tight quote-max open succeeded");
    assertGt(shot.aUsdg, shot.qUsdg, "canonical USDG not under-stated");
}
```

Canonical log: `qUsdg 2500000000 / aUsdg 2500425321 / revertRequired 2500425321 / revertMaximum 2500000000`.

**Fix:** pass `slot0.sqrtPriceX96` into `_positionAmountsForBand` instead of `TickMath.getSqrtPriceAtTick(actualTick)`.

---

## [L-01] Last Roundtable seats to leave a lane will permanently lock the per-lane reward remainder in the accumulator reserve

### Summary

`_allocateLane` always reserves the full laneBudget even when available `% activeCount` stays in rewardRemainder instead of `rewardPerSeatIndex`. Seats are only credited `rewardPerSeatIndex` deltas. When the last seats close or roll away, remainder is not flushed to anyone and future allocations see `activeCount == 0` and send new budget to the platform, leaving the old remainder as an unclaimable reserve.

### Root Cause

[OxleyRoundtableManagerR2.sol L775–L780](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L775-L780)

```solidity
uint256 available = laneBudget + lane.rewardRemainder;
uint256 perSeat = available / lane.activeCount;
lane.rewardRemainder = available % lane.activeCount;
lane.rewardPerSeatIndex += perSeat;
return (laneBudget, 0); // full budget reserved
```

Credit path ([L796–L799](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L796-L799)) only moves rewardPerSeatIndex - baseline. Close/roll ([L803–L815](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L803-L815)) decrements activeCount without paying remainder. Empty-lane later swaps hit [L773](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/src/OxleyRoundtableManagerR2.sol#L773) and redirect new budget to the platform.


### Internal Pre-conditions

- A lane had `activeCount >= 2` so a nonzero remainder can exist (`remainder < activeCount`).

- All seats on that lane later close or roll while remainder is still nonzero.


### Attack Path

1. Three seats are active; a 10-unit allocation yields perSeat = 3, remainder = 1, reserve += 10 (as in [testEqualSeatRemainderIsCarriedWithoutOverAllocation](https://github.com/greenpeez/NBA-audit/blob/main/packages/contracts/test/R2RewardLane.t.sol#L42-L55)).

2. All three collect (3+3+3 = 9 credited) and close. remainder = 1 remains; roundtableReserve still holds 1.

3. Later swaps on an empty lane redirect new budget to the platform. The leftover 1 never becomes roundtableClaimable.

No attacker profit; this is a lock. Remaining seats before the last exit can inherit remainder on the next allocate (available = budget + remainder), which is dust concentration rather than an external drain.

### Impact

At most `activeCount `- 1 wei USDG (6 decimals) per lane per empty-out. Twenty lanes × many pools is still sub-cent unless a lane had an enormous activeCount at the last modulo. Funds are backed `(_requirePoolManagerBacked still holds)` but unclaimable, so `claimBacking` can show expected claims above what anyone can pull from that remainder.

Live testnet outstanding swap claims were 1_334_738 units; remainder dust is negligible beside that.

### PoC

How to run:

```bash
FOUNDRY_PROFILE=r2 forge test --match-test testEqualSeatRemainderIsCarriedWithoutOverAllocation -vv
```

then close the three seeded seats and read roundtableReserve.

```solidity
function testRemainderOrphanedWhenLaneEmpties() public {
    (, bytes32 poolId) =
        _launch("R2 Rem Orphan", "R2REMO", keccak256("r2-rem-orphan"), OxleyCoinFactoryR2.IncentiveMode.None);
    _seedActiveCountForGas(poolId, 0, 3);
    vm.prank(address(hook));
    roundtable.allocate(
        poolId,
        IOxleyRoundtableManagerR2.AllocationPath({
            coinIsCurrency0: true,
            actualStartTick: -390_000,
            actualEndTick: -390_000,
            startSqrtPriceX96: TickMath.getSqrtPriceAtTick(-390_000),
            endSqrtPriceX96: TickMath.getSqrtPriceAtTick(-390_000)
        }),
        10
    );
    (uint64 activeCount,,, uint256 rewardIndex, uint256 remainder) = roundtable.laneState(poolId, 0);
    assertEq(activeCount, 3);
    assertEq(rewardIndex, 3);
    assertEq(remainder, 1);
    assertEq(accumulator.roundtableReserve(poolId), 10);
    // After real seats collect 3 each and close, reserve left == 1 with no claimable beneficiary.
}
```

