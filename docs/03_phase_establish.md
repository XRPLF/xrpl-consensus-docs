# ESTABLISH Phase

In the `ESTABLISH` phase, validators agree on the exact transaction set and close time for the next ledger. Each starts from the position it took when it closed its open ledger, a proposed transaction set and a close time. On each tick of the consensus timer it updates that position: it adds the transactions enough of its [trusted peers](unl.md) include, drops the ones enough of them exclude, and moves its close time toward theirs. The round reaches the `ACCEPTED` phase once enough peers settle on one position.

## Terminology

* **Transaction set**: An immutable collection of transactions, held as a Merkle tree keyed by transaction ID (a `SHAMap`). Its hash depends only on which transactions it holds, not the order they were added, so any two validators holding the same transactions compute the same hash. A position references its set by that hash, two validators agree exactly when their hashes match, and finding disputes is a tree diff over the IDs. Our set is a snapshot of our open transactions taken at close, plus vote pseudo-transactions on flag ledgers.
* **Raw close time**: A validator's own close time, from its wall clock plus network drift correction, unrounded. `raw_close_times` collects these per round: our own, plus each peer's, taken only from that peer's initial position (sequence 0). Kept for drift analysis, not the close-time vote (which uses the rounded current positions).
* **Disputed transaction**: A transaction that is in one validator's set but not another's. Each one is voted in or out during the `ESTABLISH` phase until the network converges.
* **Acquired sets**: The transaction sets the node currently holds, its own plus any fetched from peers, so it can compare positions and build disputes. A position names its set only by hash. A node that holds the hash but not the set fetches it from peers on demand. `expose` publishes our set so peers can fetch it this way.
* **Converge percent**: This round's elapsed establish time as a percentage of the previous round's duration (floored at `av_min_consensus_time`). 
* **Avalanche**: `xrpld`'s name for the rising agreement threshold described in the overview. Each disputed transaction tracks its own avalanche level.

## Overview

**Resolving disputes.** A dispute is a transaction that some validators include in their set and others leave out. A validator votes yes to keep it in its set or no to drop it, and re-votes each tick. It keeps the transaction only while enough of its trusted peers also include it. The bar for keeping it rises as the round runs, a ramp `xrpld` calls the **avalanche**, so a transaction that can't hold enough support is dropped from the set, and the sets converge on the transactions everyone agrees on. How high the bar sits is set by **converge percent**: this round's elapsed establish time as a percentage of the previous round's duration (or 5 seconds, whichever is larger). The higher it climbs, the larger the share of peers that must vote yes to keep a transaction in the set:

| Converge percent | Yes-share to keep a transaction |
|---|---|
| under 50% | over 50% |
| 50% to 85% | over 65% |
| 85% to 200% | over 70% |
| 200% and over | over 95% |

The bar climbs through these levels as converge percent rises, but it will not step up to the next level until at least two ticks have passed at the current level. For example, if the previous round took 6 seconds:

* 3 seconds in is 50% converge, so the bar steps from 50% to 65%.
* Near 5 seconds it reaches 85% (bar 70%).
* By 12 seconds it is at 200% (bar 95%).

A transaction that only 60% of peers want is kept in the set while the bar is 50%, but dropped once converge crosses 50% and the bar reaches 65%, so weakly-supported transactions fall away and the sets close in on agreement.

**Agreeing on the close time.** Alongside the transaction set, validators agree on when the ledger closed. Each validator rounds its own close time to a coarse step shared by the round, the **close-time resolution** (a value like 10 seconds), and votes for the rounded result. Rounding to the same step lets validators whose clocks differ by a few seconds vote for one time. The avalanche governs this vote too, on a track separate from the disputes': a rounded value must clear the same rising bar to be in the running, and the close time is settled once 75% of the participants back it. One difference from disputes: the close-time bar steps up on converge percent alone, without waiting two ticks at each level. If no value gathers enough votes, the close time stays unsettled and the round keeps waiting. The ledger records its close time as not agreed only once the validators settle on no agreed time, deriving it from the previous ledger (its close time plus one second).

For example, take eight validators rounding to a 30-second resolution. For 8 participants a value needs 4 votes (50%) to be the running pick and 6 (75%) to settle:

| Tick | votes for 16:00:00 | votes for 16:00:30 | Pick (≥ 4) | Settled (≥ 6) |
|---|---|---|---|---|
| 1 | 5 | 3 | 16:00:00 | no, 5 < 6 |
| 2 | 8 | 0 | 16:00:00 | yes, 8 ≥ 6 |

Between the ticks every validator adopts the running pick as its own vote, so the three on 16:00:30 move over and the tally crosses the settle bar. Had the votes stayed split instead, the candidate bar would have climbed toward 95% as the round ran (needing all 8), until nothing cleared it and the round kept waiting.

The resolution is fixed when the round opens, derived from the previous ledger, so every validator uses the same one. It is one of a fixed ladder of bin sizes, 10, 20, 30, 60, 90, or 120 seconds (starting at 30), re-chosen each round and adapting asymmetrically: it coarsens fast, one step after any round that failed to agree on a close time, and tightens slowly, one step at every eighth ledger while rounds keep agreeing. At the ends of the ladder it holds: it never coarsens past 120 seconds or tightens below 10. Rounding each close time to the nearest multiple of the current bin is what lets clocks a few seconds apart land on one value.

**Leaving the phase.** The round leaves the `ESTABLISH` phase for the `ACCEPTED` phase once we are not waiting on lagging validators, enough validators agree on the transaction set, and the close time is settled. If the network moves on without us or the round runs too long, the transaction set condition counts as met even without our genuine agreement, so it never stalls forever. The close time must still be settled to reach the `ACCEPTED` phase.

## Position

A position is a validator's current stance in the round: the transaction set it thinks the next ledger should contain, together with its close time. Every validator holds its own position and tracks the latest position of each trusted peer. The round converges when these positions come to share the same transaction set.

A position carries a sequence number. The opening position of a round is sequence 0, and each revision increments it, so a higher sequence from a validator always supersedes its lower ones. A reserved sequence value marks a validator bowing out of the round.

```
Position:
    prev_ledger   = Hash of the prior ledger this position builds on
    tx_set        = Hash of the transaction set proposed for the next ledger
    close_time    = When this validator closed the ledger (its clock at close)
    seq           = 0 for the opening position, increments on each revision
    node_id       = Which validator holds this position
    seen_time     = When this position was last taken or updated
```

## Round result

Closing the open ledger creates `result`, our stance for this round, discarded when the next round starts.

```
Result:
    tx_set     = Our transaction set
    position   = Our position
    disputes   = Disputed transactions this round, keyed by transaction
    compares   = Peer sets we have already compared this round
    round_time = How long this round has been establishing
    proposers  = Count of peers proposing this round
    state      = The round's outcome (set during establish)
```

## Closing the open ledger

The `ESTABLISH` phase begins by closing the open ledger. `close_ledger()` is invoked from the `OPEN` phase's `should_close` decision (see `02_phase_open.md`). It:

* Transitions the round to the `ESTABLISH` phase and resets the establish counters and timer.
* Records our own raw close time, the unrounded time we consider the ledger closed.
* Builds our opening position: snapshots our open transactions into the transaction set we propose, paired with our close time (`on_close`, see below).
* Makes our transaction set fetchable by peers, and broadcasts our position if we are in proposing consensus mode.

At this point some peers may have already sent their positions for this round.

* For each peer position whose transaction set we hold, we record every transaction we disagree on as a dispute to be voted on during the `ESTABLISH` phase.

```python
network_adjusted_time = Network adjusted time
phase = Current consensus round phase
mode = Consensus mode
previous_ledger = The prior ledger this round builds on
open_ledger = Our live, in-progress ledger that transactions accumulate in as they arrive
raw_close_times = Per-round record of raw close times (our own plus each peer's initial one), self feeds next round's local close time (02_phase_open.md)
result = Our stance this round (see Round result), absent until close
acquired_sets = Transaction sets we currently hold (ours and any fetched from peers)
current_peer_positions = Trusted peers' current positions for this round
peer_unchanged_counter = Consecutive ESTABLISH phase iterations with no peer changing a disputed-transaction vote
establish_counter = Number of times we ran the ESTABLISH phase code this round

def close_ledger():
    phase = ESTABLISH
    raw_close_times[self] = network_adjusted_time
    peer_unchanged_counter = 0
    establish_counter = 0

    result = on_close(network_adjusted_time)   # builds result.tx_set and result.position
    result.round_time = 0

    if result.tx_set not in acquired_sets:
        acquired_sets.add(result.tx_set)
        expose(result.tx_set)

    if mode == PROPOSING:
        broadcast(result.position)

    for peer_position in current_peer_positions:
        if peer_position.tx_set in acquired_sets:
            create_disputes(peer_position.tx_set)
```

## Taking our position

`close_ledger` belongs to the generic consensus engine, which is written to run independently of any particular ledger. Everything tied to the XRP Ledger is delegated to an adaptor, and `on_close` is the adaptor callback that turns our open ledger into a transaction set and an opening position. The split keeps the core consensus algorithm separate from `xrpld` specifics: how transactions are stored, fee and amendment voting, and the negative UNL.

`on_close`:

* Re-applies held transactions. To *apply* a transaction is to execute it against the open ledger, updating account balances and other state. The node holds transactions it accepted but could not apply when they first arrived (fee below the open ledger's current fee, or waiting on an earlier transaction from the same account), and retries them here. Those that now succeed become part of the open ledger.
* Assembles our transaction set: the open ledger's transactions, plus fee, amendment, or negative-UNL vote pseudo-transactions on the periodic [flag ledgers](flag_ledger.md) when proposing. The set is a fixed snapshot of the still-changing open ledger, identified by its hash, which is what lets validators agree by comparing hashes.
* Takes our opening position (sequence 0) over that set, stamped with our close time.

When it holds the correct last closed ledger, `on_close` also registers the proposed set with the censorship detector, a diagnostic that flags transactions repeatedly left out of accepted ledgers. It does not affect our position (see [censorship_detector.md](censorship_detector.md)).

```python
def on_close(network_adjusted_time):
    apply_held_transactions(open_ledger)
    txs = open_ledger.transactions
    if mode == PROPOSING:
        txs += flag_ledger_votes()
    tx_set = transaction_set(txs)

    position = Position(
        prev_ledger = previous_ledger.id,
        tx_set = tx_set.hash,
        close_time = network_adjusted_time,
        seq = 0,
        node_id = self,
        seen_time = network_adjusted_time)

    return Result(tx_set = tx_set, position = position)
```

## Creating disputes

`create_disputes` compares our transaction set against a peer's. For every transaction the two differ on, it records a dispute and notes each peer's vote: whether that transaction is in the peer's set. The `ESTABLISH` phase then resolves these votes round by round, validators adding or dropping transactions until the sets match.

`create_disputes(other_tx_set)` runs once per peer set we hold: at close for the positions we already have, and again each time we acquire a new peer set during the `ESTABLISH` phase. It:

* Skips a set it has already compared this round, and skips entirely if the peer's set is identical to ours.
* Takes the symmetric difference: the transactions in exactly one of the two sets.
* For each such transaction, creates a dispute recording our own vote and every held peer's vote. A peer's vote changing resets `peer_unchanged_counter`.
* Relays the disputed transaction to peers, so a peer that lacks it can fetch and reconsider it.

`update_disputes` re-applies a peer's votes across the current disputes when its set changes, creating any not-yet-compared disputes first. It runs on a new position from a peer, or when we adopt a set a peer holds.

```python
Dispute:
    tx            = The disputed transaction
    our_vote      = Whether the transaction is in our set
    peer_votes    = Per peer: whether the transaction is in that peer's set
    avalanche     = This dispute's avalanche level and tick count
    our_unchanged = Ticks since we last changed our vote

def create_disputes(other_tx_set):
    if other_tx_set in result.compares:
        return

    result.compares.add(other_tx_set)

    if other_tx_set == result.tx_set:
        return

    for tx in symmetric_difference(result.tx_set, other_tx_set):
        if tx in result.disputes:
            continue
        
        dispute = Dispute(tx, our_vote = tx in result.tx_set)
        
        for peer_position in current_peer_positions:
            if peer_position.tx_set in acquired_sets:
                peer_vote = tx in peer_position.tx_set
                if dispute.peer_votes.get(peer_position.node_id) != peer_vote:
                    dispute.peer_votes[peer_position.node_id] = peer_vote
                    peer_unchanged_counter = 0
        
        relay(tx)
        result.disputes[tx] = dispute


def update_disputes(node, tx_set):
    if tx_set not in result.compares:
        create_disputes(tx_set)

    for dispute in result.disputes:
        peer_vote = dispute.tx in tx_set
        if dispute.peer_votes.get(node) != peer_vote:
            dispute.peer_votes[node] = peer_vote
            peer_unchanged_counter = 0
```

## Receiving a peer's position

Peer positions arrive between ticks, as trusted validators broadcast and revise theirs. `receive_peer_position` validates each one, stores it as the peer's current position, and reacts to it. It runs in the `OPEN` phase too, before we have closed. The dispute steps apply only once `result` exists.

* It ignores the position if the round has already reached the `ACCEPTED` phase, the position builds on a different previous ledger, or the peer has already bowed out this round.
* It ignores the position if its sequence is no higher than the peer's current one.
* On a bow-out, it drops the peer's votes from every dispute, removes the peer from the current positions, and marks the peer dead so its later positions this round are ignored.
* Otherwise it stores the position as the peer's current one, recording the peer's raw close time from its opening position (sequence 0).
* It ensures we hold the peer's transaction set: if we already have it, it records the peer's votes on our disputes. Otherwise it fetches the set, and the votes are recorded when it arrives.

```python
dead_nodes = Peers that have bowed out this round

def receive_peer_position(peer_position):
    peer = peer_position.node_id

    if phase == ACCEPTED:                                   # round already done
        return
    if peer_position.prev_ledger != previous_ledger.id:     # builds on a different ledger
        return
    if peer in dead_nodes:                                  # already bowed out this round
        return

    current = current_peer_positions[peer]
    if current is not none and peer_position.seq <= current.seq:
        return                                              # not newer than what we hold

    if peer_position.is_bow_out():
        if result exists:
            for dispute in result.disputes:
                dispute.peer_votes.remove(peer)
        current_peer_positions.remove(peer)
        dead_nodes.add(peer)                                # ignore its later positions this round
        return

    current_peer_positions[peer] = peer_position
    if peer_position.seq == 0:                              # the peer's opening position
        raw_close_times.peers[peer_position.close_time] += 1

    if peer_position.tx_set in acquired_sets:
        if result exists:
            update_disputes(peer, peer_position.tx_set)
    else:
        fetch(peer_position.tx_set)                         # its votes are recorded when it arrives
```

When the fetch completes, `got_tx_set` adds the set to our acquired sets and, if the round has closed, records the votes of every peer already proposing it.

```python
def got_tx_set(tx_set):
    if phase == ACCEPTED:                          # round already done
        return
    if tx_set in acquired_sets:                    # already have it
        return
    acquired_sets.add(tx_set)

    if result exists:                              # before close there are no disputes yet
        for peer_position in current_peer_positions:
            if peer_position.tx_set == tx_set.hash:
                update_disputes(peer_position.node_id, tx_set)
```

## The ESTABLISH tick

While the round is in the `ESTABLISH` phase, the timer calls `phase_establish` on each tick, the `ESTABLISH`-phase counterpart of `phase_open` (see `01_phase_dispatch.md`). Each tick advances the round timer, revises our position toward our peers, and checks whether the round can conclude.

* It counts the tick (`establish_counter`, and `peer_unchanged_counter`, which resets when any peer changes a dispute vote). These gate when the round can conclude: the avalanche steps up only after enough ticks at a level, and a run of unchanged ticks marks the disputes stalled.
* It advances `result.round_time` and recomputes `converge_percent`. The avalanche bar and the minimum and maximum consensus times all rise with elapsed establish time, so advancing the clock is what moves the round toward a decision.
* It returns early until `ledger_min_consensus` has passed, so every validator can propose before anyone revises.
* It revises our position: `drop_stale_peers` drops peers we have not heard from, `resolve_disputes` updates our votes and set, `determine_close_time` settles the close time, and if the set or close time changed `revise_position` commits the new position (below).
* It moves to `ACCEPTED` only when we are not pausing for laggards (`should_pause`), the transaction set has consensus (`have_consensus`), and the close time is settled (`have_close_time_consensus`). Otherwise it stays in `ESTABLISH` and the next tick rechecks.
* On accept it carries `prev_proposers` and `prev_round_time` to the next round and hands `result` to `on_accept`, which builds and validates the new ledger (see `04_phase_accepted.md`).

```python
phase = Current consensus phase
result = Our stance this round
current_peer_positions = Trusted peers' latest positions
establish_counter = Establish iterations this round
peer_unchanged_counter = Establish iterations since the last dispute-vote change
clock_now = Current monotonic clock value
round_start = Monotonic clock value at close
converge_percent = This round's establish duration as a percent of the previous round's
prev_round_time = The previous round's establish duration
prev_proposers = The previous round's proposer count
ledger_min_consensus = 1.95 seconds
av_min_consensus_time = 5 seconds
have_close_time_consensus = Whether peers agree on the close time
now = Current network-adjusted time
propose_interval = 12 seconds

def phase_establish():
    establish_counter += 1
    peer_unchanged_counter += 1

    result.round_time = clock_now - round_start
    result.proposers = count(current_peer_positions)
    converge_percent = result.round_time * 100 / max(prev_round_time, av_min_consensus_time)

    if result.round_time < ledger_min_consensus:
        return

    drop_stale_peers()
    set_changed = resolve_disputes()
    close_time = determine_close_time()
    position_stale = result.position.seen_time <= now - propose_interval
    if set_changed or close_time != as_close_time(result.position.close_time) or position_stale:
        revise_position(close_time)

    if should_pause() or not have_consensus():
        return
    if not have_close_time_consensus:
        return

    # consensus reached on the transaction set and the close time
    prev_proposers = count(current_peer_positions)
    prev_round_time = result.round_time
    phase = ACCEPTED
    on_accept(result)
```

The `drop_stale_peers`, `resolve_disputes`, `determine_close_time`, and `revise_position` steps above are this spec's split, for readability. In `xrpld` they are not separate functions: all run inside one function, `updateOurPositions`, called once per tick.

`drop_stale_peers` removes any peer we have not heard a fresh position from within `propose_freshness`, pulling its votes from every dispute.

```python
propose_freshness = 20 seconds

def drop_stale_peers():
    for peer_position in current_peer_positions:
        if peer_position.seen_time <= now - propose_freshness:
            for dispute in result.disputes:
                dispute.peer_votes.remove(peer_position.node_id)
            current_peer_positions.remove(peer_position)
```

`revise_position`:

* Updates our position to the new transaction set and close time, and increments its sequence number.
* The first time we hold this set, exposes it so peers can fetch it.
* Also on first hold, records the votes of any peer already proposing this set.
* Broadcasts the position if proposing.

Counting a peer's vote needs its set. The first time a set enters our acquired sets, whether fetched or built here, we record the votes of every peer already proposing it. A set we already held has had this done.

```python
def revise_position(close_time):
    result.position.tx_set = result.tx_set.hash       # adopt the new set
    result.position.close_time = close_time
    result.position.seen_time = now
    if not result.position.is_bow_out():              # a bowed-out validator freezes its sequence
        result.position.seq += 1
    if result.tx_set not in acquired_sets:
        acquired_sets.add(result.tx_set)
        if not result.position.is_bow_out():
            expose(result.tx_set)
        for peer_position in current_peer_positions:
            if peer_position.tx_set == result.tx_set.hash:
                update_disputes(peer_position.node_id, result.tx_set)
    if not result.position.is_bow_out() and mode == PROPOSING:
        broadcast(result.position)
```

## Voting on disputes

`resolve_disputes` updates our vote on every dispute with `update_vote` and adopts the resulting set, returning whether our set changed. `update_vote` keeps or drops a transaction by weighing peer support against `needed_weight`, the avalanche threshold. 

```python
def resolve_disputes():
    new_tx_set = copy(result.tx_set)
    for dispute in result.disputes:
        if update_vote(dispute):
            if dispute.our_vote:
                new_tx_set.add(dispute.tx)
            else:
                new_tx_set.remove(dispute.tx)
                
    if result.tx_set == new_tx_set:
        return False
    
    result.tx_set = new_tx_set
    return True

def update_vote(dispute):
    yes = count of dispute.peer_votes that are true
    no  = count of dispute.peer_votes that are false

    disagree = no if dispute.our_vote else yes   # peers voting against our current vote
    if disagree == 0:
        return False

    required_pct = needed_weight(dispute.avalanche, converge_percent, av_min_rounds)
    
    if mode == PROPOSING:
        yes_pct = (yes + (1 if dispute.our_vote else 0)) * 100 / (yes + no + 1)
        new_vote = yes_pct > required_pct
    else:
        new_vote = yes > no

    if new_vote == dispute.our_vote:
        dispute.our_unchanged += 1       # held our vote another tick
        return False

    dispute.our_unchanged = 0            # changed our vote
    dispute.our_vote = new_vote
    return True
```

## The avalanche

`needed_weight` is the avalanche: the yes-share a transaction (or close time) currently needs. It climbs one level at a time as `converge_percent` rises, stepping up only when converge reaches the next level. Disputes pass `av_min_rounds`, so a level holds for at least two ticks. The close time passes 0 and steps up as soon as converge allows. Each caller keeps its own avalanche state, so disputes and the close time climb the ladder independently. In `xrpld`, `getNeededWeight` is pure, returning the next level for the caller to store. Here `needed_weight` advances the level in place.

```python
av_min_rounds = 2

# converge percent that unlocks each level, and the yes-share it then requires
avalanche_ladder = [(0, 50), (50, 65), (85, 70), (200, 95)]

Avalanche:
    level = 0    # current level in avalanche_ladder
    ticks = 0    # ticks spent at this level

def needed_weight(avalanche, converge_percent, min_ticks):
    avalanche.ticks += 1
    if avalanche.level + 1 < len(avalanche_ladder) and avalanche.ticks >= min_ticks:
        next_converge, next_share = avalanche_ladder[avalanche.level + 1]
        if converge_percent >= next_converge:        # step up one level, no further
            avalanche.level += 1
            avalanche.ticks = 0
    _, share = avalanche_ladder[avalanche.level]
    return share
```

## Determining the close time

`determine_close_time` tallies the rounded close times (peers' and ours, if proposing) and picks the agreed one, settling `have_close_time_consensus` at the `av_ct_consensus_pct` threshold. `as_close_time` does the rounding.

```python
av_ct_consensus_pct = 75
close_resolution = This round's close-time bin size, in seconds
close_time_avalanche = Avalanche state for the close-time vote

def determine_close_time():
    if current_peer_positions is empty:
        have_close_time_consensus = True
        return as_close_time(result.position.close_time)   # alone: our own close time stands

    votes = {}
    for peer_position in current_peer_positions:
        votes[as_close_time(peer_position.close_time)] += 1
    participants = count(current_peer_positions)
    if mode == PROPOSING:
        votes[as_close_time(result.position.close_time)] += 1
        participants += 1

    candidate_pct = needed_weight(close_time_avalanche, converge_percent, 0)
    candidate_threshold = max((participants * candidate_pct + candidate_pct / 2) / 100, 1)
    consensus_threshold = max((participants * av_ct_consensus_pct + av_ct_consensus_pct / 2) / 100, 1)

    consensus_close_time = none
    have_close_time_consensus = False
    best = candidate_threshold
    for time, count in votes ordered by time:
        if count >= best:
            consensus_close_time = time
            best = count
            if best >= consensus_threshold:
                have_close_time_consensus = True
    return consensus_close_time

def as_close_time(raw):                       # raw is seconds since epoch
    if raw == 0:                              # unset close time passes through unchanged
        return raw
    biased = raw + close_resolution / 2       # bias by half a bin so the truncation rounds to nearest
    return biased - (biased mod close_resolution)
```

## Pausing for laggards

A validator can close ledgers faster than the network confirms them. An accepted ledger becomes **fully validated** once a quorum of trusted validators sign it.

`ahead` is how far our working ledger sits past our last fully validated ledger. A **validation** is the signed message a validator broadcasts announcing the ledger it accepts as correct. Each trusted validator is classified by its most recent validation:

* **caught up**: a fresh validation at or beyond our ledger.
* **laggard**: a fresh validation, but on an earlier ledger.
* **offline**: no fresh validation within `validation_freshness`.

`should_pause` does not wait when waiting is pointless: we are not ahead, nothing lags, we are not a validator, we have never fully validated a ledger, or the round has already run past `ledger_max_consensus`.

Otherwise it compares how many trusted validators are caught up against a required fraction, and waits if too few are. The required fraction is set by the **pause phase**, which is derived from `ahead`, not from the tick count: `pause_phase = (ahead - 1) mod (max_pause_phase + 1)`. So one ledger ahead is phase 0, two is phase 1, and so on up to `max_pause_phase`, then it wraps back to 0. At phase 0 it asks only for the quorum fraction to be caught up - each higher phase asks for more, up to all validators at the top. The wrap means a node stuck far ahead keeps cycling through the thresholds instead of holding at the strictest one.

```python
validation_freshness = 20 seconds
ledger_max_consensus = 15 seconds
max_pause_phase = 4
valid_ledger_seq = Sequence of our last fully validated ledger
am_validator = Whether we are configured as a validator
have_validated = Whether we have ever fully validated a ledger
trusted_keys = Signing keys of our trusted validators (our UNL)
negative_unl = Trusted validators currently voted offline (the negative UNL)
now = Current network-adjusted time

def should_pause():
    ahead = previous_ledger.seq - min(valid_ledger_seq, previous_ledger.seq)
    total = count(trusted_keys)

    laggards = 0
    offline  = 0
    for key in trusted_keys:
        v = latest_validation(key)
        if v is none or now >= v.seen_time + validation_freshness:
            offline += 1
        elif v.seq < previous_ledger.seq:
            laggards += 1
        # else: caught up

    if (ahead == 0 or laggards == 0 or total == 0
            or not am_validator or not have_validated
            or result.round_time > ledger_max_consensus):
        return False

    caught_up = total - laggards - offline
    pause_phase = (ahead - 1) mod (max_pause_phase + 1)

    # required caught-up fraction: quorum_ratio at pause_phase 0, reaching 1 at pause_phase max_pause_phase
    quorum_ratio = quorum() / total
    required = quorum_ratio + (1 - quorum_ratio) * pause_phase / max_pause_phase
    return caught_up / total < required
```

[`quorum`](unl.md#quorum) is the number of trusted validators whose signatures fully validate a ledger, the same threshold the network applies to declare full validation. In normal operation it is 80% of the effective UNL, the trusted set minus any validator on the negative UNL (voted offline so the round can still reach quorum without it), and never less than 60% of the full trusted set.

```python
def quorum():
    unl_size = count(trusted_keys)
    effective_size = count(key in trusted_keys if key not in negative_unl)
    return max(ceil(effective_size * 0.8), ceil(unl_size * 0.6))
```

## Reaching consensus

`have_consensus` is the transaction-set check the `ESTABLISH` tick runs: have enough trusted validators settled on our transaction set, or has the round otherwise run its course? It counts how many peers hold our position and records the round's outcome as `result.state`:

| Outcome | Meaning | Reaches the `ACCEPTED` phase? |
|---|---|---|
| **`Yes`** | Enough proposers hold our transaction set, or the round can make no further progress (disputes deadlocked, or we are alone past the time limit). | Yes |
| **`MovedOn`** | We never agreed, but enough proposers have finished this round and validated the next ledger without us. | Yes |
| **`Expired`** | The round ran far too long. | Yes, but held back at first (see below) |
| **`No`** | Not yet. | No: revise and recheck next tick |

`Expired` is held back at first: a round that expires before every avalanche level has had its `av_min_rounds` (before `establish_counter` reaches `avalanche levels x av_min_rounds`) keeps trying, since the round was merely slow, not genuinely stuck.

On a real expiry we **bow out** of proposing: we announce our withdrawal so peers stop counting our position toward agreement, and switch to `OBSERVING` mode, where we no longer broadcast or revise a position but keep tracking peers and following the round to its end. Having given up on reaching agreement, we let the round move to the `ACCEPTED` phase on the transaction set we currently hold, the one named by our position, even though not enough peers settled on it. This bow-out is `leave_consensus`, defined below.

The core test is a supermajority: counting ourselves when proposing, at least `min_consensus_pct` (80%) of the proposers must hold our transaction set. Nothing is decided before `ledger_min_consensus`. If fewer than three quarters of the previous round's proposers are here, we do not rush. Once the round passes `ledger_max_consensus`, a node with no peer sharing its position declares consensus rather than hang.

```python
min_consensus_pct = 80
ledger_abandon_factor = 10
ledger_abandon_consensus = 120 seconds
av_stalled_rounds = 4

def have_consensus():
    agree = 0
    disagree = 0
    for peer_position in current_peer_positions:
        if peer_position.tx_set == result.position.tx_set:
            agree += 1
        else:
            disagree += 1

    finished = proposers_finished()        # peers that validated the next ledger, moving on without us
    stalled = (have_close_time_consensus and result.disputes
               and all(dispute_stalled(d) for d in result.disputes))

    result.state = check_consensus(agree + disagree, agree, finished, stalled)

    if result.state == NO:
        return False
    if result.state == EXPIRED:
        if establish_counter < len(avalanche_ladder) * av_min_rounds:
            return False                   # ran slow, not truly stuck, keep trying
        leave_consensus()                  # bow out of proposing, switch to observing
    return True

def check_consensus(proposers, agree, finished, stalled):
    if result.round_time <= ledger_min_consensus:
        return NO  # too early to decide

    # proposers dropped off sharply: give them time rather than rush
    if proposers < prev_proposers * 3 / 4 and result.round_time < prev_round_time + ledger_min_consensus:
        return NO

    if stalled:
        return YES  # disputes locked at a supermajority, nothing will move

    reached_max = result.round_time > ledger_max_consensus

    if enough_support(agree, proposers, mode == PROPOSING, reached_max):
        return YES  # enough proposers hold our set
    if enough_support(finished, proposers, False, reached_max):
        return MOVED_ON  # enough proposers moved on to the next ledger without us

    abandon = clamp(prev_round_time * ledger_abandon_factor, ledger_max_consensus, ledger_abandon_consensus)
    if result.round_time > abandon:
        return EXPIRED  # ran far past the limit with no agreement
    return NO

def enough_support(supporters, proposers, count_self, reached_max):
    if proposers == 0:  # nobody shares our position
        return reached_max  # alone past the max: yes anyway
    if count_self:  # include our own vote when proposing
        supporters += 1
        proposers += 1
    return supporters * 100 / proposers >= min_consensus_pct

def leave_consensus():
    if mode == PROPOSING:
        if result exists and not result.position.is_bow_out():
            result.position.seq = BOW_OUT       # reserved bow-out sequence
            result.position.seen_time = now
            broadcast(result.position)          # announce we are leaving
        mode = OBSERVING
```

`leave_consensus` is our own bow-out, called only on a real expiry. It marks our position with the reserved bow-out sequence, broadcasts it so peers stop counting us, and drops us to `OBSERVING` for the rest of the round. This mirrors the peer side: the same `BOW_OUT` marker we set here is the one `receive_peer_position` reacts to when a peer leaves.

`proposers_finished` counts the trusted validators that have moved past this round: those whose latest validation is for a ledger built on top of our previous ledger, not the previous ledger itself. They have finished the round and validated the next ledger, and the `MovedOn` outcome weighs them against the proposer count.

```python
def proposers_finished():
    finished = 0
    for key in trusted_keys:
        v = latest_validation(key)
        if v is not none and v.ledger is a strict descendant of previous_ledger:
            finished += 1
    return finished
```

A disputed transaction is **stalled** when its vote can no longer move: 
- its avalanche has reached the top level and held there for `av_min_rounds`, 
- the votes have settled (we or our peers unchanged for `av_stalled_rounds`), 
- and the transaction rests at a clear supermajority for or against. 

Only when the close time is settled and every dispute is stalled does `have_consensus` treat the round as stalled, declaring Yes so a stubborn minority cannot hold the round open.

```python
def dispute_stalled(dispute):
    # the avalanche can still rise, or has not held at the top for av_min_rounds
    if dispute.avalanche.level < len(avalanche_ladder) - 1 or dispute.avalanche.ticks < av_min_rounds:
        return False
        
    # when proposing, we have not held our own vote for av_min_rounds
    if mode == PROPOSING and dispute.our_unchanged < av_min_rounds:
        return False
        
    # votes still moving: both we and our peers changed within av_stalled_rounds
    if peer_unchanged_counter < av_stalled_rounds and (mode == PROPOSING and dispute.our_unchanged < av_stalled_rounds):
        return False

    # clear supermajority for or against the transaction
    yes = count of dispute.peer_votes that are true
    no  = count of dispute.peer_votes that are false
    support = (yes + (1 if mode == PROPOSING and dispute.our_vote else 0)) * 100
    total = no + yes + (1 if mode == PROPOSING else 0)
    if total == 0:
        return False
    weight = support / total
    return weight > min_consensus_pct or weight < (100 - min_consensus_pct)
```

## C++ reference

| Pseudocode | C++ |
|---|---|
| **`Position`** | the `ConsensusProposal` type (`ConsensusProposal.h`). A trusted peer's arrives wrapped as `RCLCxPeerPos` |
| **`Result`** | `ConsensusResult` (`ConsensusTypes.h`) |
| **`close_ledger`** | `Consensus::closeLedger` |
| `open_ledger` | `ApplicationImp::openLedger_` |
| `raw_close_times` | `Consensus::rawCloseTimes_` |
| `result` | `Consensus::result_` |
| `acquired_sets` | `Consensus::acquired_` |
| `current_peer_positions` | `Consensus::currPeerPositions_` |
| `peer_unchanged_counter` | `Consensus::peerUnchangedCounter_` |
| `establish_counter` | `Consensus::establishCounter_` |
| **`on_close`** | `RCLConsensus::Adaptor::onClose` |
| **`create_disputes`** | `Consensus::createDisputes` |
| **`update_disputes`** | `Consensus::updateDisputes` |
| `Dispute` | `DisputedTx` (`DisputedTx.h`) |
| **`receive_peer_position`** | `Consensus::peerProposal` -> `peerProposalInternal` |
| `dead_nodes` | `Consensus::deadNodes_` |
| **`got_tx_set`** | `Consensus::gotTxSet` |
| **`phase_establish`** | `Consensus::phaseEstablish`. On the phase change it hands `result` to `adaptor_.onAccept` (`RCLConsensus::Adaptor::onAccept`) |
| `round_start` | `ConsensusResult::roundTime` start (a `ConsensusTimer`) |
| `converge_percent` | `Consensus::convergePercent_` |
| `ledger_min_consensus` | `ConsensusParms::ledgerMinConsensus` |
| `av_min_consensus_time` | `ConsensusParms::avMinConsensusTime` |
| `have_close_time_consensus` | `Consensus::haveCloseTimeConsensus_` |
| `propose_interval` | `ConsensusParms::proposeINTERVAL` |
| **`drop_stale_peers`** | part of `Consensus::updateOurPositions` |
| `propose_freshness` | `ConsensusParms::proposeFRESHNESS` |
| **`revise_position`** | part of `Consensus::updateOurPositions` |
| **`resolve_disputes`** | the dispute loop in `Consensus::updateOurPositions` |
| **`update_vote`** | `DisputedTx::updateVote` |
| **`needed_weight`** | `getNeededWeight` (`ConsensusParms.h`) |
| `av_min_rounds` | `ConsensusParms::avMinRounds` |
| `avalanche_ladder` | `ConsensusParms::avalancheCutoffs` |
| **`determine_close_time`** | close-time tally in `Consensus::updateOurPositions` (the `closeTimeVotes` map, setting `haveCloseTimeConsensus_`) |
| **`as_close_time`** | `Consensus::asCloseTime` -> `roundCloseTime` (`LedgerTiming.h`) |
| `av_ct_consensus_pct` | `ConsensusParms::avCtConsensusPct` |
| `close_resolution` | `Consensus::closeResolution_` |
| `close_time_avalanche` | `Consensus::closeTimeAvalancheState_` |
| **`should_pause`** | `Consensus::shouldPause` |
| `validation_freshness` | `ValidationParms::validationFRESHNESS` |
| `ledger_max_consensus` | `ConsensusParms::ledgerMaxConsensus` |
| `max_pause_phase` | free const `kMaxPausePhase` (in `Consensus::shouldPause`) |
| `valid_ledger_seq` | `LedgerMaster::validLedgerSeq_` (via `Adaptor::getValidLedgerIndex`) |
| `am_validator` | `RCLConsensus::Adaptor::validator()` (= `validatorKeys_.keys.has_value()`) |
| `have_validated` | `LedgerMaster::haveValidated()` (= `!validLedger_.empty()`) |
| `trusted_keys` | `ValidatorList::trustedSigningKeys_` (via `getQuorumKeys`) |
| `negative_unl` | `ValidatorList::negativeUNL_` |
| **`quorum`** | `ValidatorList::quorum` |
| **`have_consensus`** | `Consensus::haveConsensus` |
| **`check_consensus`** | free function `checkConsensus` (`Consensus.h`) |
| **`enough_support`** | part of the free function `checkConsensus` (`Consensus.h`) |
| **`leave_consensus`** | `Consensus::leaveConsensus` |
| `min_consensus_pct` | `ConsensusParms::minConsensusPct` |
| `ledger_abandon_factor` | `ConsensusParms::ledgerAbandonConsensusFactor` |
| `ledger_abandon_consensus` | `ConsensusParms::ledgerAbandonConsensus` |
| `av_stalled_rounds` | `ConsensusParms::avStalledRounds` |
| **`proposers_finished`** | `Consensus::proposersFinished` |
| **`dispute_stalled`** | `DisputedTx::stalled` |

## Generic engine and the adaptor

The steps here (closing, disputes, position revision, the consensus checks) run in the generic engine, which works only with opaque transaction-set hashes and positions. The XRP Ledger-specific parts, assembling the real transaction set in `on_close` and building the ledger in `on_accept`, live in the adaptor ([code_organization.md](code_organization.md#the-rcl-binding)).
