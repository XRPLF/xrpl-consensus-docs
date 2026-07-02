# The UNL and Trusted Validators

A node does not trust every validator on the network, only a chosen set: its Unique Node List, or UNL. 
In practice the operator does not name validators one by one. 
It configures one or more validator list publishers, and the node trusts the validators those lists name. 
Those validators are the node's trusted validators, recomputed once per round as the next round starts (`ValidatorList::updateTrusted`, see [Starting the Next Round](05_start_consensus_round_again.md#beginning-consensus)).

Two derived sets matter. The **negative UNL** is a subset of the UNL, recorded on the ledger, voted temporarily disabled for being unreliable. 
The **effective UNL** is the **full UNL** minus the negative UNL.

A trusted validator's input is used in several places:

* **positions.** a position counts toward agreement in the `ESTABLISH` phase if its signer is on the **full UNL** (see [establish phase](03_phase_establish.md)).
* **closing the open ledger.** when deciding to close, the `OPEN` phase counts over the **full UNL** how many trusted validators have already closed or fully validated the previous ledger (see [open phase](02_phase_open.md)).
* **the fork choice.** the preferred prior ledger is computed from all trusted validations over the **full UNL** (see [preferred_ledger.md](preferred_ledger.md)).
* **full validation.** a ledger is fully validated only on a quorum of trusted full validations over the **effective UNL** (see [quorum](#quorum)).
* **pausing for laggards.** the `ESTABLISH` phase brake compares **full UNL** counts against the **effective UNL** quorum (see [pausing for laggards](03_phase_establish.md#pausing-for-laggards)).

## Quorum

The quorum is the bar a ledger must clear to count as fully validated.

`ValidatorList::quorum()` returns the cached quorum value, last set by `calculateQuorum` during [`updateTrusted`](05_start_consensus_round_again.md#beginning-consensus). Each round `updateTrusted` rebuilds the trusted master keys, counts how many of them are on the negative UNL, and computes:

```python
def calculate_quorum(unl_size, effective_size):   # effective_size = unl_size minus trusted keys on the negative UNL
    if quorum_override is set:          # --quorum on the command line, logged as potentially unsafe
        return quorum_override
    if publisher lists are configured and too many are unavailable:
        return INFINITY                # quorum unreachable: the node will not fully validate on an incomplete trust set
    return max(ceil(effective_size * 0.8), ceil(unl_size * 0.6))
```

This is the same value that [03's](03_phase_establish.md#pausing-for-laggards) `quorum()` reads. 

## Negative UNL

The negative UNL keeps the network progressing when a few trusted validators are down. An unreliable validator is voted onto it, which drops it from the effective UNL, so the quorum shrinks and the validators still online can reach it. The list is capped at 25% of the UNL, and the 60% floor bounds how far the quorum can fall. Validators are voted on and off through `ttUNL_MODIFY` pseudo-transactions, on the flag-ledger cadence (every 256th ledger, see [flag_ledger.md](flag_ledger.md)). The whole mechanism is amendment-gated (`featureNegativeUNL`).

### On the ledger

The negative UNL lives in one ledger object (`keylet::negativeUNL`). Every ledger carries it, not just flag ledgers: a non-flag ledger copies the object unchanged from its parent, so a node that has just synced sees the current negative UNL without waiting for the next flag ledger. `sfDisabledValidators` holds the committed master keys, each stamped with the sequence it was disabled at, and `ledger.negativeUNL()` returns that set. Pending fields `sfValidatorToDisable` and `sfValidatorToReEnable` hold a change that has been agreed but not yet committed. A `sfValidatorToReEnable` is only set when the negative UNL is non-empty.

### Adjusting the quorum

At the start of each round `begin_consensus` loads the previous ledger's negative UNL into the validator list (`setNegativeUNL`), and `updateTrusted` subtracts those keys from the effective UNL before computing the quorum (see [Quorum](#quorum)). The counting side matches: when `checkAccept` tallies trusted full validations for a ledger, it first drops validations signed by negative-UNL validators (`negativeUNLFilter`), so a disabled validator counts toward neither the quorum nor the tally reaching it. The 60% floor still holds, so disabling validators lowers the bar only so far.

### Voting a validator on or off

On the round that builds a flag ledger, a proposing node casts its negative-UNL votes through `NegativeUNLVote::doVoting`, which adds `ttUNL_MODIFY` pseudo-transactions to its proposed set. The votes come from a reliability score: for each trusted validator, `buildScoreTable` counts how many of its validations over the last flag-ledger period (the 256 ancestors read from the skip list) agree with the node's own chain. A validation counts toward a validator's score only when it names the same ledger the scoring node holds at that sequence. A node builds the table only if it has the full period of history and issued at least 90% of that period's validations itself, so the score is not drawn from an incomplete record.

`findAllCandidates` then selects, by score:

* **to disable**: a validator with validations for under 50% of the period's ledgers, not already disabled, and not newly trusted (a newly trusted validator is exempt for two flag-ledger periods).
* **to re-enable**: a disabled validator that has recovered to over 80%, or one that has dropped off the UNL entirely.

Disabling is capped: a node proposes a disable only while the negative UNL holds under 25% of the UNL. At most one disable and one re-enable are voted per flag ledger, each picked deterministically from the candidates by the parent ledger hash, so every node proposes the same one.

### Taking effect

A `ttUNL_MODIFY` is valid only in a flag ledger (`Change::applyUNLModify`). Applying it records the change in the pending field, not the committed set. `Ledger::updateNegativeUNL`, run while a flag ledger is built, commits the pending fields: it appends a pending disable to `sfDisabledValidators` and removes a pending re-enable, then clears both. A change therefore takes effect at the flag ledger after the one that voted it: set as pending at flag ledger N, committed at flag ledger N+256. Counting the flag-ledger period the reliability score already needs, a validator that goes offline takes up to two flag-ledger intervals to appear on the list. 

## C++ reference

| Concept | C++ |
|---|---|
| Quorum value | `ValidatorList::quorum_`, `ValidatorList::quorum` |
| Quorum formula | `ValidatorList::calculateQuorum` |
| Command-line override | `ValidatorList::minimumQuorum_` |
| Recompute trust and quorum each round | `ValidatorList::updateTrusted` |
| Load negative UNL into the validator list | `ValidatorList::setNegativeUNL` |
| Drop disabled validators from a validation count | `ValidatorList::negativeUNLFilter` |
| Count trusted full validations for the quorum | `LedgerMaster::checkAccept`, `Validations::getTrustedForLedger` |
| Negative UNL on the ledger | `Ledger::negativeUNL`, `keylet::negativeUNL` |
| Pending disable / re-enable | `Ledger::validatorToDisable`, `Ledger::validatorToReEnable` |
| Commit pending changes | `Ledger::updateNegativeUNL` |
| Cast negative-UNL votes | `NegativeUNLVote::doVoting` |
| Reliability score table | `NegativeUNLVote::buildScoreTable` |
| Select candidates | `NegativeUNLVote::findAllCandidates` |
| Exempt newly trusted validators | `NegativeUNLVote::newValidators` |
| Add the pseudo-transaction | `NegativeUNLVote::addTx` |
| Apply the pseudo-transaction | `Change::applyUNLModify` |
| Flag ledger / voting ledger | `Ledger::isFlagLedger`, `Ledger::isVotingLedger` |
