# Starting the Next Round

When `do_accept` finishes building and adopting the new ledger (see [ACCEPTED Phase](04_phase_accepted.md)), the same background job calls `end_consensus`, which starts the next consensus round. This is the loop that turns one round into a continuously advancing ledger chain: `end_consensus` settles the node's sync state and hands off to `begin_consensus`, which rebuilds the trusted validator set and quorum, then opens a fresh round back in the `OPEN` phase.

## Terminology

* **Operating mode**: The node's own view of how connected and how synced it is: `DISCONNECTED`, `CONNECTED`, `SYNCING`, `TRACKING`, or `FULL`. Only a `FULL` node proposes. The heartbeat ([Phase Dispatch](01_phase_dispatch.md)) and `end_consensus` both adjust it.
* **Validating**: Whether the node will sign full validations this round. It requires validator keys, a previous ledger at or above the reboot anti-equivocation floor, no amendment block, and a non-expired validator list. Recomputed every round in `pre_start_round`.
* **Proposing**: `validating` and synced (operating mode `FULL`). A proposing node runs the round in `PROPOSING` consensus mode and pushes its own position. Otherwise it runs in `OBSERVING` mode and only watches.
* **Trusted validator set**: The UNL validators the node currently trusts, rebuilt at the start of each round from the validator list and the previous ledger's [negative UNL](unl.md#negative-unl). Its size sets the round's `quorum`.
* **Network-closed ledger**: The ledger the network as a whole considers most recently closed, determined by polling peers' reports and trusted validations. The new round builds on top of it.

## Overview

The `ACCEPTED` phase ends by building a ledger and seeding the next open ledger on top of it, but no round is running on that ledger yet. Starting the next round does that: it rebuilds the trusted validator set and recomputes the quorum, decides whether the node will propose, and resets all per-round state so the engine is back in the `OPEN` phase, ready to collect transactions.

In order, the work is to:

* Clear the closed ledger announced by peers a round behind, so stale notes do not linger.
* Settle the operating mode toward synced (`TRACKING`, then `FULL`) from the node's sync state.
* Reload the negative UNL and recompute the trusted validator set and the round's quorum.
* Decide whether the node validates this round, and whether it also proposes.
* Choose the previous ledger to build on, acquiring the network's preferred one if it differs.
* Reset all per-round state and set the phase back to `OPEN`.
* Replay proposals peers already sent for the new ledger, and if more than half of last round's proposers already have one, run a timer tick at once instead of waiting for the next heartbeat.

Once this is done the node is back in the `OPEN` phase ([OPEN Phase](02_phase_open.md)), accumulating transactions for the next ledger, and the cycle repeats.

Most of this runs outside the generic engine: `end_consensus` and `begin_consensus` are in `NetworkOPs`, and `pre_start_round` in the adaptor. Together they settle the sync state, rebuild trust and quorum, and decide whether to propose. Only the last step, resetting per-round state and reopening the round (`start_round` and `start_round_internal`), is the generic engine.

## Ending the round

`end_consensus` runs as the second half of the accept job, right after `do_accept`. 

First it forgets stale ledger announcements. Each peer broadcasts the hash of the ledger it most recently closed in a status message (the [Announcing the ledger](04_phase_accepted.md#announcing-the-ledger) step of that peer's own round), which the node caches. Any peer still announcing the ledger we just superseded (the parent of the one we closed) is a round behind, so the node clears its cached note of that peer's closed ledger until the peer announces a current one.

Then it asks what the network considers closed. `check_last_closed_ledger` polls peers' reported ledgers and trusted validations to find the network's preferred closed ledger, and reports whether that differs from the one we just closed. If there is no network view yet, `end_consensus` returns without starting a round.

When we do agree with the network on the closed ledger, `end_consensus` promotes the operating mode toward synced:

* From `CONNECTED` or `SYNCING` to `TRACKING`, unless the node still needs to acquire a ledger from the network first.
* From `CONNECTED` or `TRACKING` to `FULL`, if the current ledger closed recently enough that the node is plausibly caught up.

This is the mode that `pre_start_round` reads a moment later to decide whether the node may propose. Finally it calls `begin_consensus`.

```python
operating_mode        # node's sync state: DISCONNECTED, CONNECTED, SYNCING, TRACKING, FULL
need_network_ledger   # still waiting to acquire a ledger from the network before going live
closed_ledger         # the ledger we just built and adopted (see ACCEPTED phase)

def end_consensus():
    dead_ledger = closed_ledger.parent_hash       # the ledger before the one we just closed
    for peer in active_peers:
        if peer.closed_ledger_hash == dead_ledger:
            peer.cycle_status()                   # forget this peer's stale closed-ledger report

    network_closed, ledger_changed = check_last_closed_ledger()   # network's closed ledger, from peer reports and trusted validations
    if network_closed == 0:                       # no network view yet
        return

    # promote sync state only while we agree with the network on the closed ledger
    if operating_mode in (CONNECTED, SYNCING) and not ledger_changed and not need_network_ledger:
        operating_mode = TRACKING
    if operating_mode in (CONNECTED, TRACKING) and not ledger_changed and closed_recently():
        operating_mode = FULL

    begin_consensus(network_closed)
```

## Beginning consensus

`begin_consensus` rebuilds the trusted validator set and [quorum](unl.md#quorum) the round will run against, then starts it.

It builds on the ledger `do_accept` just closed, the parent of the current open ledger (see [Seeding the next open ledger](04_phase_accepted.md#seeding-the-next-open-ledger)), which becomes the round's previous ledger. If the node does not hold that parent, it has jumped ledgers, so it drops from `FULL` to `TRACKING` and gives up starting the round.

It then recomputes the trusted validator set and quorum:

* It loads the negative UNL recorded in the previous ledger, so disabled validators do not count toward the quorum.
* It recomputes the trusted validator set from the validators it has been hearing from, which sets the round's quorum.
* If the trusted set changed, it tells the validation store and the amendment table, so both track the current validators.

```python
def begin_consensus(network_closed):
    prev_ledger = ledger_by_hash(current_open_ledger.parent_hash)   # the ledger do_accept just closed
    if prev_ledger is None:                       # jumped ledgers, we don't hold the LCL
        if operating_mode == FULL:
            operating_mode = TRACKING
        return

    set_negative_unl(prev_ledger.negative_unl)    # disabled validators recorded in that ledger
    changes = update_trusted(seen_validators, prev_ledger.close_time)   # rebuild the trusted set and quorum
    if changes.added or changes.removed:
        validations.trust_changed(changes.added, changes.removed)
        amendment_table.trust_changed(quorum_keys())

    proposing = pre_start_round(prev_ledger, changes.added)
    start_round(now, network_closed, prev_ledger, changes.removed, proposing)
```

## Deciding whether to propose

`pre_start_round` runs once per round, just before the engine starts. It recomputes whether the node may sign full validations this round (`validating`) and returns whether it should also propose.

A node may validate only if all hold:

* It has validator keys configured.
* The previous ledger is at or above `max_disallowed_ledger`, the reboot anti-equivocation floor (see [ACCEPTED phase](04_phase_accepted.md#broadcasting-the-validation)). After a restart this blocks signing until the chain has advanced past the last ledger the node persisted.
* It is not amendment-blocked, meaning its software is not too old to understand an enabled amendment.
* Its validator list is configured and not expired. A node will not validate against a stale UNL.

Whether it also proposes adds one more requirement: it must be synced, meaning operating mode `FULL`. A node that is validating but not yet synced enters the round to observe and learn the validated ledger, without pushing a position of its own.

Two side effects ride along here, unrelated to the propose decision: it tells ledger acquisition that a new round has started, and it notifies negative-UNL voting of any newly trusted validators.

```python
validating            # node state: will we sign full validations this round?
validator_keys        # our validator keys, if configured
max_disallowed_ledger # reboot anti-equivocation floor (see ACCEPTED phase)
operating_mode        # node's sync state

def pre_start_round(prev_ledger, now_trusted):
    validating = (validator_keys is set
                  and prev_ledger.seq >= max_disallowed_ledger
                  and not blocked())              # amendment-blocked?

    if validating and not standalone and validators.count != 0:
        if validators.expires is None or validators.expires < now:
            validating = False                    # stale validator list

    synced = operating_mode == FULL               # only a synced node proposes

    inbound_transactions.new_round(prev_ledger.seq)        # side effect: ledger acquisition
    if now_trusted:
        negative_unl_vote.new_validators(prev_ledger.seq + 1, now_trusted)   # side effect

    return validating and synced                  # proposing
```

## Starting the round

`start_round` is the consensus engine's entry point for a new round. It picks the starting consensus mode and makes sure it is building on the right previous ledger.

It first sets the previous round's timing values, seeded from defaults on the instance's first round.

Validators that just lost trust have their buffered positions dropped, so nothing they sent earlier replays into the new round.

The mode starts as `PROPOSING` or `OBSERVING` per `pre_start_round`. If the network's preferred previous ledger (`prev_ledger_id`) differs from the one we closed, the node acquires and builds on it. Failing that, it starts on ours in `WRONG_LEDGER` mode, and the heartbeat's `check_ledger` switches on a later tick once it holds the right one (see [Phase Dispatch](01_phase_dispatch.md)).

```python
first_round           # true until the first round this process runs
prev_round_time       # how long the previous round took to converge
prev_close_time       # our own close time, carried into the next round
raw_close_times       # this round's close-time votes (self plus peers)

def start_round(now, prev_ledger_id, prev_ledger, now_untrusted, proposing):
    if first_round:
        prev_round_time = LEDGER_IDLE_INTERVAL    # no prior round to measure
        prev_close_time = prev_ledger.close_time
        first_round = False
    else:
        prev_close_time = raw_close_times.self    # our close time from the round just finished

    for n in now_untrusted:
        recent_peer_positions.pop(n)              # drop buffered positions from dropped validators

    mode = PROPOSING if proposing else OBSERVING

    if prev_ledger.id != prev_ledger_id:          # network closed a ledger we don't hold
        acquired = acquire_ledger(prev_ledger_id)
        if acquired is not None:
            prev_ledger = acquired
        else:
            mode = WRONG_LEDGER 

    start_round_internal(now, prev_ledger_id, prev_ledger, mode)
```

## Resetting per-round state

`start_round_internal` is where the round actually begins. It clears the previous round's state and sets the phase back to `OPEN`, records the chosen mode, adopts the previous ledger, and computes the new ledger's close-time resolution (see [Determining the close time](03_phase_establish.md#determining-the-close-time)).

Last, it replays proposals peers already buffered for the new ledger. If more of them already have a position than half the previous round's proposer count, the node is likely lagging, so it runs a timer tick ([Phase Dispatch](01_phase_dispatch.md)) at once instead of waiting for the next heartbeat.

```python
phase                 # current consensus phase
previous_ledger       # the ledger this round builds on
close_resolution      # close-time bin size for the ledger about to be built
curr_peer_positions   # peers' positions this round
prev_proposers        # how many proposers the previous round had

def start_round_internal(now, prev_ledger_id, prev_ledger, mode):
    phase = OPEN                                   # back to the OPEN phase
    set_mode(mode)
    previous_ledger = prev_ledger
    result = None                                 # clear last round's stance
    converge_percent = 0
    have_close_time_consensus = False
    open_time.reset(monotonic_now())              # start timing the OPEN phase
    curr_peer_positions.clear()
    raw_close_times.clear()
    dead_nodes.clear()

    close_resolution = next_close_resolution(     # see ESTABLISH phase: Determining the close time
        prev_ledger.close_time_resolution, prev_ledger.close_agree, prev_ledger.seq + 1)

    playback_proposals()                          # replay positions peers already sent for this ledger
    if len(curr_peer_positions) > prev_proposers / 2:
        timer_entry(now)                          # likely lagging: tick now rather than wait for the next heartbeat
```

## C++ reference

| Pseudocode | C++ |
|---|---|
| **`end_consensus`** | `NetworkOPsImp::endConsensus`. Mode changes go through `setMode` |
| `need_network_ledger` | `NetworkOPsImp::needNetworkLedger_` |
| `cycle_status` | `Peer::cycleStatus` |
| `check_last_closed_ledger` | `NetworkOPsImp::checkLastClosedLedger` (sets `networkClosed`, returns whether the ledger changed) |
| `closed_recently` | the `now < parentCloseTime + 2 * closeTimeResolution` test on the current ledger |
| **`begin_consensus`** | `NetworkOPsImp::beginConsensus` |
| `update_trusted` | `ValidatorList::updateTrusted` (it computes the quorum) |
| `set_negative_unl` | `ValidatorList::setNegativeUNL` |
| `now` | `getTimeKeeper().closeTime()` |
| **`pre_start_round`** | `RCLConsensus::Adaptor::preStartRound`, which sets the `validating_` member |
| `validator_keys` | `RCLConsensus::Adaptor::validatorKeys_` |
| `max_disallowed_ledger` | `Application::getMaxDisallowedLedger` (the getter read here) |
| `blocked` | `NetworkOPs::isBlocked` |
| `validators.expires` | `ValidatorList::expires` |
| **`start_round`** | `Consensus::startRound` |
| `first_round` | `Consensus::firstRound_` |
| `acquire_ledger` | the adaptor's `acquireLedger` |
| `recent_peer_positions` | `recentPeerPositions_` |
| **`start_round_internal`** | `Consensus::startRoundInternal` |
| `next_close_resolution` | the free function `getNextLedgerTimeResolution` |
| `playback_proposals` | `playbackProposals` |
| `open_time` | the monotonic `openTime_` stopwatch |
