# ACCEPTED Phase

The `ACCEPTED` phase is a consensus round's final phase. The node builds the next ledger and adopts it as its **last closed ledger**, the base it now builds on, immediately and on its own.

Adopting that ledger is a local decision, made from the validator's own view of the round. There is no guarantee that network as a whole landed on the same ledger in the previous phase. It may not have some validators accept on a give-up condition instead of genuine agreement, and a fork can leave nodes on different chains.
So each validator broadcasts a **validation** of the exact ledger it accepted, and once a quorum of trusted validators have validated the same ledger it is **fully validated**, the settled ledger for its sequence that the node never reverts.

## Terminology

* **Validation**: A validator's signed statement that it has accepted a particular ledger. It carries the ledger's hash and sequence number and the hash of the transaction set the ledger contains. Validation is either:
    * **Full**: broadcast while the validator is proposing. It asserts full participation in the round and counts toward fully validating the ledger.
    * **Partial**: broadcast while not proposing, such as after bowing out. It does not count toward fully validating the ledger.
* **Last closed ledger**: The ledger this node has most recently built and adopted as the base for its next round. The node adopts it when it accepts.
* **Fully validated ledger**: A ledger this node treats as final: it holds a quorum of trusted full validations for the ledger, and the ledger descends from the node's current fully validated ledger. Each node decides this locally from the validations it has received, so it usually lags the last closed ledger.

## Overview

A round reaches the `ACCEPTED` phase when the consensus engine decides the `ESTABLISH` phase should stop, either because enough trusted validators share our transaction set or on a give-up condition (the network moved on, the round ran too long, or the disputes deadlocked). Each is an outcome of `check_consensus` in [Reaching consensus](03_phase_establish.md#reaching-consensus). The close-time vote has resolved by this point too, possibly to "no agreed time" (see [Determining the close time](03_phase_establish.md#determining-the-close-time)).

Unlike the `OPEN` and `ESTABLISH` phases, the `ACCEPTED` phase is not driven by the heartbeat timer. 
A tick during the `ESTABLISH` phase enters it by calling `on_accept`, which schedules one background job: `do_accept`, then `end_consensus` to start the next round. 
Every other entry into the consensus engine (timer ticks, incoming peer positions) runs under one lock. This job runs without it, which is safe because once the round reaches the `ACCEPTED` phase the engine freezes the round's `result` until the next round starts, and the job itself starts it.

This phase runs in the adaptor (`RCLConsensus::Adaptor`) and `LedgerMaster`, not the generic engine (`Consensus<Adaptor>`), whose part of the round ended when `phase_establish` called `on_accept`. The one generic piece still involved is `Validations`: our validation is recorded there, and the quorum check reads it for the count.

Even when the `ESTABLISH` phase did not reach agreement from our perspective, we build and adopt our own ledger in the `ESTABLISH` phase. Catching a wrong ledger is left to the preferred-ledger logic that `check_ledger` consults on every tick (see [Phase Dispatch](01_phase_dispatch.md)). If our ledger is not the one the network prefers, the node switches on a later heartbeat, not here. The fork choice is specified in [preferred_ledger.md](preferred_ledger.md).

At a high level, `do_accept` runs these steps, each detailed in its own section below:

```python
def do_accept():
    build the next ledger from the agreed tx set and close time
    announce built to peers
    if validating and the round succeeded and the sequence is allowed:
        sign and broadcast our validation of built
    try to advance our fully validated ledger
    open the next round's ledger on built
    adopt built as our last closed ledger
    if we took part and the round succeeded:
        nudge our close-time clock toward the network
```

Whether we broadcast a validation, and of what kind, depends on the state that ended the previous phase:

| State | What we broadcast | Counts toward fully validating it? |
|---|---|---|
| **`Yes`** | Full: we were still proposing. | Yes |
| **`Expired`** | Partial: we had bowed out to observing. | No |
| **`MovedOn`** | Nothing: the round failed (`consensusFail`, the network reached consensus and moved on to the next ledger without us), so we broadcast no validation and skip the close-time clock adjustment. | n/a |
 
Two entry points drive the fully validated ledger forward: `do_accept` runs the steps above once per round, and each arriving peer validation funnels into the same quorum check, `check_for_quorum` ([Incoming validations](#incoming-validations)).

## Accepting Ledger

```python
# inputs for this round
result            # our stance this round: tx set, position, disputes, state (see [Round result](03_phase_establish.md#round-result))
previous_ledger   # the prior ledger this round built on
close_resolution  # this round's close-time bin size (see [Determining the close time](03_phase_establish.md#determining-the-close-time))
mode              # this round's consensus mode: PROPOSING, OBSERVING, or WRONG_LEDGER (see [Phase Dispatch](01_phase_dispatch.md))
raw_close_times   # ours plus each peer's initial (seq-0) close time (see [Raw close time](03_phase_establish.md#terminology))

# node state (persists across rounds)
validating             # whether we are validating this round (hold keys, in sync, not blocked)
valid_ledger           # last fully validated ledger we hold (a full ledger)
closed_ledger          # our last closed ledger, the base the next round builds on
last_quorum_ledger     # highest (id, seq) seen reaching a quorum of trusted full validations, may be ahead of valid_ledger
building_ledger        # sequence of the ledger being built this round (cleared once built)
quorum                 # validations needed to fully validate a ledger (~80% of the trusted UNL, negative-UNL adjusted)
last_validation_time   # sign time of our most recent validation, kept strictly increasing
close_offset           # damped drift correction added to our wall clock to track network time
largest_validated_seq  # largest sequence we have validated recently: the in-memory anti-equivocation floor
largest_validated_when # when we last advanced largest_validated_seq
validation_set_expires # 10 min, the floor resets if no newer sequence is recorded for this long
max_disallowed_ledger  # reboot anti-equivocation floor: greatest ledger persisted before (re)start (on-disk counterpart of largest_validated_seq)
max_ledger_difference  # tolerance for the reboot-floor assert
local_txs              # transactions submitted to this node, not yet in a ledger
queued_txs             # transactions waiting in the transaction queue (TxQ), ordered by fee, for a future ledger
```

### Building the ledger

Accepting the ledger begins by turning the round's result into a concrete ledger, built from the agreed transaction set and close time:

- **Resolve the close time.** If the round settled on a close time, round it to the shared bin, ensuring its after parent's close time. If it instead settled on no agreed time, derive the close time from the parent (one second later) and mark it "not agreed".
- **Order the transactions.** Sort the agreed set by a salt derived from the set hash, so every node produces the same order yet no one can arrange it in advance. Drop any that fail to decode.
- **Apply to the parent.** Produce the new ledger from the ordered transactions.
- **Keep the leftover transactions.** Transactions that fail now but might succeed later stay in `retriable_txs`, which seeds the next round.

The build also updates the ledger's **skip list**: an on-ledger index of ancestor hashes (the most recent 256 ledgers, plus every 256th ledger further back) that each new ledger extends with its parent's hash. It backs `hashOfSeq(ledger, seq)`, the ancestor-by-sequence lookup that the compatibility check (`areCompatible`, behind `LedgerMaster::isCompatible`) relies on later in the accepted phase.

On [flag ledgers](flag_ledger.md#the-flag-ledger-cadence), the build also refreshes the negative UNL, applying any pending validator disable or re-enable, which adjusts `quorum` for the following rounds.

After the build, the transaction queue is pruned of transactions the new ledger has made stale (`TxQ::processClosedLedger`). 

The built ledger is also cached by hash (`storeLedger`), so it can be retrieved later, both when we later fully validate it and when peers request it.

```python
def do_accept():
    # resolve the agreed close time
    if result.position.close_time == 0:                  # no close time agreed this round
        close_time = previous_ledger.close_time + 1s     # "agreed to disagree"
        close_time_correct = False
    else:
        close_time = max(round_to_resolution(result.position.close_time, close_resolution),
                         previous_ledger.close_time + 1s)  # round to bin, stay after the parent
        close_time_correct = True

    # order the agreed txns (salted, deterministic) and drop any that fail to decode
    retriable_txs = canonical_tx_set(salt = result.tx_set.hash)
    for tx in result.tx_set:
        if decodes(tx):
            retriable_txs.insert(tx)

    # apply onto the parent, unapplied txns come back in retriable_txs to retry next round
    built, retriable_txs = apply_transactions(previous_ledger, retriable_txs, close_time, close_time_correct)
    ...
```

`buildLCL` also has a replay branch: it can rebuild one specific historical ledger from saved transaction data instead of from the consensus set. That data is populated only by the `--replay` admin startup mode. The pseudocode omits the branch.

### Announcing the ledger

With the ledger built, `do_accept` tells peers which ledger it now holds. This is a status message, not a validation. Peers use it for:

- **Catch-up.** It advertises which ledger we hold and the range we can serve, so a peer filling a gap knows it can fetch that history from us.
- **Last-resort fork choice.** A peer that has no trusted validations yet (just starting, or fully out of sync) falls back to counting which ledger its connected peers report, and follows the most common. Our announcement is one such count, whether or not the peer trusts us. 
Once the peer has any trusted validations, those decide and the announcement no longer affects its fork choice.

```python
def do_accept():
    ...
    # announce to peers which ledger we now hold (a status message, not a validation)
    event = ACCEPTED_LEDGER if mode != WRONG_LEDGER else LOST_SYNC
    broadcast_status(event, built)
    ...
```

### Updating the censorship detector

`do_accept` then updates the censorship detector: it tracks transactions we proposed that keep being left out of accepted ledgers and warns when one has waited too long. It is purely diagnostic and never changes consensus state, so the pseudocode skips it. See [censorship_detector.md](censorship_detector.md) for what it tracks and when it runs.

### Checking the ledger is compatible

Before broadcasting a validation, `do_accept` confirms the ledger it built does not conflict with the chain that is already fully validated (the ledgers that have reached a quorum of trusted full validations as we have seen them). 
It keeps validating this round only if `built` does not conflict with either anchor we track:

1. **`valid_ledger`**, a full ledger we hold. `built` must lie on the same chain, ancestor or descendant. Holding the full ledger lets us compare in either direction.
2. **`last_quorum_ledger`**, known only by id and sequence. `built` must not fork from it at or above its height. Lacking the ledger itself, we cannot walk below it, so a `built` older than it passes unchecked.

`last_quorum_ledger` can run ahead of `valid_ledger`. A ledger can reach quorum before we hold it in full, so it catches forks the validated ledger alone would miss. 

If there is conflict, `validating` is set to `false`.

```python
def do_accept():
    ...
    if validating:
        # keep validating only if built conflicts with neither anchor
        validating = on_same_chain(built, valid_ledger) \
            and not_forking_from(built, last_quorum_ledger)
    ...


def on_same_chain(candidate, anchor):            # anchor is a full ledger we hold: compare either way
    if anchor is None:
        return True
    if anchor.seq < candidate.seq:
        h = candidate.ancestor(anchor.seq)       # candidate's ancestor at the anchor's height
        return h is None or h == anchor.id
    if anchor.seq > candidate.seq:
        h = anchor.ancestor(candidate.seq)       # walk the anchor back to the candidate's height
        return h is None or h == candidate.id
    return candidate.id == anchor.id


def not_forking_from(candidate, anchor):         # anchor is only (id, seq): forward-only
    if anchor is None:
        return True
    if candidate.seq > anchor.seq:
        h = candidate.ancestor(anchor.seq)
        return h is None or h == anchor.id
    if candidate.seq == anchor.seq:
        return candidate.id == anchor.id
    return True                                  # candidate older than the anchor: cannot check
```

### Broadcasting the validation

After the compatibility check, `do_accept` issues our validation for `built`, but only when:

- we are still validating (the compatibility check did not clear `validating`),
- the `ESTABLISH` phase did not fail (`result.state != MOVED_ON`),
- the anti-equivocation floor permits this sequence.

The anti-equivocation floor refuses to sign at or below a sequence we have already validated, so we avoid issuing two conflicting validations for the same height. It is a test-and-set that records the sequence on success. If 10 minutes (`validation_set_expires`) pass without recording a newer sequence, that record expires and the floor resets to 0.

We then sign `built` with a strictly increasing timestamp, so no two of our validations share a sign time. We record the validation locally first, counting our own vote, and pre-register its hash so that when a peer relays it back to us, we drop it rather than reprocessing our own validation. Then we broadcast it to peers and publish it to local subscribers.

A validation carries:

```
Validation:
    ledger_hash    = Hash of the ledger this validator accepts
    ledger_seq     = That ledger's sequence number
    consensus_hash = Hash of the transaction set the ledger contains
    full           = True if signed while proposing (counts toward quorum), False if partial
    validated_hash = Hash of the signer's own last fully validated ledger
    sign_time      = When it was signed, strictly increasing per validator
    cookie         = Per-instance random value, tells apart two nodes signing the same key
    signer         = The validator's key (and node id)
```

On [flag ledgers](flag_ledger.md#how-votes-travel), the validation also carries the node's fee-change and amendment votes, which the network tallies across validations to enact fee and amendment changes.

```python
def do_accept():
    ...
    # broadcast our validation, if eligible
    if validating and result.state != MOVED_ON:
        # anti-equivocation floor: never sign at or below a seq we already validated
        if now() > largest_validated_when + validation_set_expires:
            largest_validated_seq = 0                  # floor expired after a long idle gap
        if built.seq > largest_validated_seq:
            largest_validated_seq = built.seq          # record the new floor
            largest_validated_when = now()
            validation_time = max(now(), last_validation_time + 1s)   # strictly increasing sign time
            last_validation_time = validation_time
            v = sign_validation(built, full=(mode == PROPOSING), validation_time)
            suppress(v)                                # ignore our own copy when peers echo it back
            record_validation(v)                       # count locally first, also runs for peer validations
            broadcast(v)                               # to all peers
            publish(v)                                 # to local subscribers
    ...
```

### Advancing the fully validated ledger

The ledger we built (and stored in `built`), is not necessarily our **fully validated ledger**. Fully validated state is a separate, quorum-based status: it usually trails the build (the validations take time to arrive), but the quorum for `built` may already be in, and we may even have fully validated a ledger newer than `built`.

A validation is just a notification: a peer telling us which ledger it accepted. A quorum of them fully-validates a ledger, but holding that ledger's contents is a separate matter we may have to fetch from peers. That is why the node tracks two things: `valid_ledger`, the highest ledger it both holds in full and has a quorum for, and `last_quorum_ledger`, the highest (id, sequence) it has a quorum for whether or not it holds it. The second can run ahead of the first when a ledger reaches quorum before we hold it.

To advance our fully validated ledger, the node adopts a ledger as `valid_ledger` and publishes it. It does this with whatever validations have arrived so far: it tries `built` first, which works only if a [quorum](unl.md#quorum) already backs it. Otherwise it looks for another ledger backed by more than a quorum of validations (strictly more, unlike the at-least-quorum test in `check_for_quorum`) and newer than our validated one, and adopts that instead, the path by which a node that built a minority ledger takes the network's. If nothing qualifies yet, our fully validated ledger advances later, as peer validations arrive through `record_validation`.

A ledger becomes our fully validated ledger only if:

- it is a sane successor of our current fully validated ledger: its close time is within a few minutes of our clock, and its sequence is not implausibly far ahead,
- it is newer than that ledger,
- a quorum of trusted full validations backs it.

These conditions include no ancestry check: the sane-successor test bounds only close time and sequence, not descent from our current fully validated ledger. The sequence must be strictly higher, so full validation never moves to an equal or lower one. A conflicting ledger at a higher sequence that reaches quorum is still adopted, re-pointing `valid_ledger` onto that branch and abandoning the ledger fully validated below it, with nothing rolled back. That requires a quorum of trusted validators to fully-validate a conflicting ledger, which sufficient UNL overlap precludes.

Adopting a ledger as fully validated also runs the downstream effects (`publish_and_advance`): it records the ledger in the validate-side fork detector, reacts to amendments (halting the node if a required amendment it does not support has gone live), and wakes the publish/fetch engine that publishes validated ledgers to clients and fetches any the node is missing. 
It also sanity-asserts the ledger is not below the reboot floor `max_disallowed_ledger` (also enforced at round start), then does bookkeeping: the validated-ledger timestamp, local-tx pruning, persistence, remote fee tracking, and an upgrade warning.

`check_for_quorum` is the shared quorum check both paths call. When a ledger crosses quorum it advances `last_quorum_ledger` and fully validates the ledger, fetching it from peers if we do not hold it. There is one ledger `check_for_quorum` does not fully validate this way: the one we are still building. We do not hold the finished ledger yet, so it would otherwise try to fetch it from peers, which is pointless for a ledger we are about to produce ourselves. So it skips that ledger, and once the build completes, `do_accept`'s advance step fully validates `built` directly (the `fully_validate(built)` call in the code below), since we already hold the just-built ledger.

Every validation passes through `record_validation`, both our own (from the previous step) and each one a peer sends. It marks the validation trusted if its signer is currently on our UNL (our own is already trusted), since only trusted validations count toward quorum, and records it. A newly recorded trusted validation then calls `check_for_quorum`. Conflicting or duplicate validations from a validator are detected and logged here (the [Byzantine behavior detector](byzantine_detector.md)) but change no consensus state.

`do_accept` also hands `built` to a fork detector that warns if the ledger we built later diverges from the one the network validates. That is diagnostic only, so the pseudocode skips it.

```python
def do_accept():
    ...
    # advance the fully validated ledger
    building_ledger = none                       # we are no longer building
    if built.seq > valid_ledger.seq:             # not already validated this far
        fully_validate(built)                   # fully validate built if a quorum backs it
        if built.seq > valid_ledger.seq:         # built was not fully validated
            # adopt another ledger the network has already fully validated, if newer
            candidate = highest_seq_over_quorum(current_trusted_validations())   # strictly more than quorum validations
            if candidate and candidate.seq > valid_ledger.seq:
                check_for_quorum(candidate.id, candidate.seq)
    ...


def record_validation(v):                   # handleNewValidation, also run for every peer validation
    if not v.trusted and on_our_unl(v.signer):   # peer validations: trusted iff signer is on our UNL
        v.trusted = True
    if add_validation(v) == CURRENT and v.trusted:
        check_for_quorum(v.id, v.seq)       # only trusted validations can fully validate a ledger


def check_for_quorum(id, seq):              # checkAccept(hash, seq)
    if seq < valid_ledger.seq:              # older than what we fully validated
        return
    val_count = trusted_full_validations(id, seq)
    if val_count >= quorum and seq > last_quorum_ledger.seq:
        last_quorum_ledger = (id, seq)      # highest ledger seen reaching quorum
    if seq == valid_ledger.seq or seq == building_ledger:   # already validated, or mid-build
        return
    ledger = held(id)
    if ledger is None:
        if valid_ledger is None and val_count >= quorum:
            check_tracking(seq)             # startup: no validated ledger yet, mark peers far from this seq as diverged
        ledger = acquire(id, seq)           # fetch from peers
    if ledger:
        fully_validate(ledger)


def fully_validate(ledger):                # checkAccept(ledger)
    if sane_successor(ledger) and ledger.seq > valid_ledger.seq \
            and trusted_full_validations(ledger) >= quorum:
        publish_and_advance(ledger)


def publish_and_advance(ledger):            # setValidLedger and the checkAccept tail
    # sanity assert, the reboot anti-equivocation floor is actually enforced at round start (gating validating)
    assert valid_ledger.seq > 0 or ledger.seq + max_ledger_difference > max_disallowed_ledger
    valid_ledger = ledger                   # adopt as our fully validated ledger

    record_validated(ledger)                # validate-side half of the built-vs-validated fork detector

    # amendments: halt if a now-live amendment we do not support has been enabled
    amendment_table.on_validated(ledger)
    if amendment_table.has_unsupported_enabled():
        set_amendment_blocked()
    elif amendment_table.first_unsupported_expected():
        set_amendment_warned()

    try_advance()                           # wake the publish/fetch engine: publish validated ledgers, fetch any we lack

    # bookkeeping (not shown): validated-ledger timestamp = median of validators' sign times, drop
    # now-confirmed txns from the open pool, notify the store, persist and order-book setup on the first
    # validated ledger, remote load fee = median of validators' fee votes, periodic upgrade-version warning
```

### Seeding the next open ledger

With `built` in hand, `do_accept` opens the next round's ledger on top of it and seeds it with the transactions to reconsider.

First it adds to `retriable_txs` the disputed transactions we voted to exclude this round (skipping pseudo-transactions, which cannot be resubmitted). These get first priority next round: each was already proposed by a trusted validator, so it is the most likely to gain agreement.

The new open ledger is then built on `built` from these sources:

- `retriable_txs`, now holding both the build's apply failures and those disputed transactions,
- the transactions in the outgoing open ledger, those that arrived while the round ran,
- `local_txs`, transactions submitted directly to this node,
- `queued_txs`, transactions waiting in the transaction queue (TxQ, ordered by fee) for a future ledger.

```python
def do_accept():
    ...
    # disputed txns we voted to exclude get another chance next round
    for dispute in result.disputes:
        if not dispute.our_vote and not is_pseudo(dispute.tx):
            retriable_txs.insert(dispute.tx)

    # open the next round's ledger on built, applying retriable, local, and queued txns.
    # accept also reapplies the outgoing open ledger's own transactions
    open_ledger.accept(built, retriable_txs, local_txs, queued_txs)
    ...
```

### Switching the last closed ledger

`do_accept` records `built` as our **last closed ledger**, the base the next round builds on. The switch also re-runs the fully-validate check on `built`, a no-op here since the advance step already tried, but the underlying function is shared with other paths (startup, sync) that rely on it.

```python
def do_accept():
    ...
    closed_ledger = built          # adopt built as our last closed ledger
    ...
```

### Adjusting the close-time clock

`do_accept` ends by nudging our clock toward the network's. If we took part this round (proposing or observing) and the result of the previous phase was not `MovedOn`, `do_accept` averages the round's raw close times, ours and each peer's, and eases our close-time correction toward that average: for an offset beyond one second it steps a quarter of the way plus a fixed 3-second bias, otherwise it decays the existing correction toward zero.

```python
def do_accept():
    ...
    # nudge our close-time correction toward the network, damped so the clock eases over rather than jumps
    if mode in (PROPOSING, OBSERVING) and result.state != MOVED_ON:
        offset = average(raw_close_times) - raw_close_times.self   # how far our close time sits from the network's
        if offset > 1s:
            close_offset += (offset + 3s) / 4
        elif offset < -1s:
            close_offset += (offset - 3s) / 4
        else:
            close_offset = close_offset * 3 / 4   # within a second: decay the correction toward zero
```

## Incoming validations

Validations from other validators arrive continuously, not just during our own rounds, and are processed as they come in. Because they arrive at any time, our fully validated ledger can advance between our rounds: once enough validations for a ledger accumulate it becomes fully validated, even one we did not build ourselves, and `last_quorum_ledger` stays current.

```python
def on_peer_validation(v):                   # NetworkOPs::recvValidation
    if seen_before(v):                       # ignore echoes of validations we already hold
        return
    record_validation(v)                     # trust, store, and maybe fully validate (same path as our own)
    publish(v)                               # to local subscribers
    if v.trusted or config.relay_untrusted:
        relay(v)                             # forward to peers so the network converges
```

`record_validation`, `check_for_quorum`, and `fully_validate` are the same functions defined in [Advancing the fully validated ledger](#advancing-the-fully-validated-ledger). This path just drives them from peer messages instead of from `do_accept`.

## C++ reference

| Pseudocode | C++ |
|---|---|
| **`do_accept`** | `RCLConsensus::Adaptor::doAccept` (`RCLConsensus.cpp`) |
| `validating` | `RCLConsensus::Adaptor::validating_` |
| `valid_ledger` | `LedgerMaster::validLedger_` |
| `closed_ledger` | `LedgerMaster::closedLedger_` |
| `last_quorum_ledger` | `LedgerMaster::lastValidLedger_` |
| `building_ledger` | `LedgerMaster::buildingLedgerSeq_` |
| `quorum` | `ValidatorList::quorum_` |
| `last_validation_time` | `RCLConsensus::Adaptor::lastValidationTime_` |
| `largest_validated_seq` | `SeqEnforcer::seq_` (inside `Validations::localSeqEnforcer_`) |
| `largest_validated_when` | `SeqEnforcer::when_` (inside `localSeqEnforcer_`) |
| `validation_set_expires` | `ValidationParms::validationSetExpires` |
| `max_disallowed_ledger` | `ApplicationImp::maxDisallowedLedger_` |
| `max_ledger_difference` | `LedgerMaster::maxLedgerDifference_` |
| `local_txs` | `RCLConsensus::Adaptor::localTxs_` |
| `queued_txs` | the `TxQ` (`ApplicationImp::txQ_`) |
| **Building the ledger** | `buildLCL` -> `buildLedger` -> `buildLedgerImpl` (`BuildLedger.cpp`). Close-time rounding in `LedgerTiming.h`, negative-UNL refresh in `updateNegativeUNL`, skip list update in `updateSkipList`, multi-pass tx apply in `applyTransactions`, sealing in `setAccepted`/`setImmutable` |
| **Announcing the ledger** | the announcement is `notify` (`RCLConsensus.cpp`), a `TMStatusChange`. Peers consume it in `PeerImp::onMessage`, feeding `getPreferredLCL`'s peer-count fallback and ledger fetching |
| **Checking the ledger is compatible** | the check is `LedgerMaster::isCompatible` (`LedgerMaster.cpp`) |
| `on_same_chain` | an `areCompatible` overload (`View.cpp`), tested against `getValidatedLedger()` |
| `not_forking_from` | an `areCompatible` overload (`View.cpp`), tested against `lastValidLedger_` |
| **Broadcasting the validation** | `RCLConsensus::Adaptor::validate`. The floor is `Validations::canValidateSeq` (the `localSeqEnforcer_`), signing builds an `STValidation`, `suppress` is `HashRouter::addSuppression`, `broadcast` is `Overlay::broadcast`, `publish` is `NetworkOPs::pubValidation`. On flag ledgers the fee/amendment votes are `feeVote_->doValidation` / `AmendmentTable::doValidation` |
| **Advancing the fully validated ledger** | the advance step is `LedgerMaster::consensusBuilt` |
| `fully_validate` | `checkAccept` overload `(ledger)` |
| `check_for_quorum` | `checkAccept` overload `(hash, seq)` |
| `record_validation` | `handleNewValidation` (trust set from the UNL via `getTrustedKey`), also called from `NetworkOPs::recvValidation` |
| `sane_successor` | `canBeCurrent` |
| `trusted_full_validations` | `getTrustedForLedger` |
| `held` | `getLedgerByHash` |
| `acquire` | `InboundLedgers::acquire` |
| `check_tracking` | `Overlay::checkTracking` |
| `publish_and_advance` | the `setValidLedger` tail: floor via `getMaxDisallowedLedger`, fork detect `ledgerHistory.validatedLedger`, amendments `doValidatedLedger` / `setAmendmentBlocked`, publish/fetch `tryAdvance` -> `doAdvance` |
| **Seeding the next open ledger** | the disputes loop and `OpenLedger::accept` in `doAccept`. `queued_txs` applied via the inner `TxQ::accept`, active rules from `makeRulesGivenLedger` |
| `is_pseudo` | `isPseudoTx` |
| **Switching the last closed ledger** | `LedgerMaster::switchLCL` (which also re-runs `checkAccept(built)`) |
| **Adjusting the close-time clock** | the damped step is `TimeKeeper::adjustCloseTime`. `raw_close_times` is `rawCloseTimes` (self plus each peer's initial close time, peers weighted by count) |
| **`on_peer_validation`** | the overlay delivers a `TMValidation`, deduplicated through `HashRouter`. `on_peer_validation` is `NetworkOPs::recvValidation`, which calls `handleNewValidation` and relays per `relayUntrustedValidations` / `isTrusted` |
