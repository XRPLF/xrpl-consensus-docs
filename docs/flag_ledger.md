# Flag Ledger

A flag ledger is a ledger whose sequence is a multiple of 256 (the `kFlagLedgerInterval`). Its predicate is `Ledger::isFlagLedger()`, which tests `seq % 256 == 0`. The ledger just before a flag ledger is the voting ledger, whose sequence satisfies `seq % 256 == 255` (`Ledger::isVotingLedger()`). A validator's validation of a voting ledger carries that validator's fee and amendment votes. `xrpld` uses this cadence for the network's periodic voting: fee voting, amendment voting, and negative UNL updates.

The generic engine has no concept of a flag ledger, an amendment, a fee, or the negative UNL (its source references none of them). Everything XRP Ledger-specific, these votes included, reaches it through the adaptor ([code_organization.md](code_organization.md#the-rcl-binding)).

On the relevant ledgers the adaptor's `onClose`, which turns the open ledger into the proposed transaction set on the `OPEN` to `ESTABLISH` transition ([Taking our position](03_phase_establish.md#taking-our-position)), adds the votes to that set as pseudo-transactions. A pseudo-transaction is one the consensus process inserts itself, attributed to the zero account rather than submitted by a user.

## The flag-ledger cadence

`onClose` builds the proposed set for the next ledger, on top of `prev_ledger` (the last closed ledger), and branches on what `prev_ledger` is. 

```python
def flag_ledger_votes(prev_ledger):       # the flag_ledger_votes() placeholder from on_close
    if not (standalone or (proposing and not wrong_ledger)):
        return                            # only a proposing node adds these

    if prev_ledger.is_flag_ledger():                # prev_ledger.seq % 256 == 0, building flag + 1
        # tally fee and amendment votes from the voting ledger's validations
        validations = trusted_validations_for(prev_ledger.parent_hash, prev_ledger.seq - 1)
        if len(validations) >= quorum:
            fee_vote.do_voting(prev_ledger, validations, tx_set)         # ttFEE
            amendment_table.do_voting(prev_ledger, validations, tx_set)  # ttAMENDMENT

    elif prev_ledger.is_voting_ledger():            # prev_ledger.seq % 256 == 255, building the flag ledger
        # negative-UNL changes, derived from validation reliability
        negative_unl_vote.do_voting(prev_ledger, trusted_keys, validations, tx_set)  # ttUNL_MODIFY
```

The two cases are not symmetric. Negative-UNL pseudo-transactions go into the flag ledger itself: `Change::applyUNLModify` rejects a `ttUNL_MODIFY` unless `isFlagLedger(view().seq())` holds, so it can apply nowhere else.

Fee and amendment pseudo-transactions go into the ledger after the flag ledger, `flag + 1`, each stamped with that sequence. `applyFee` and `applyAmendment` carry no flag-ledger restriction. So negative-UNL changes land on the flag ledger, while fee and amendment changes land one ledger later.

## How votes travel

Fee and amendment votes are transmitted inside validations, which is broadcast during the `ACCEPTED` phase.
When a validator builds and validates a voting ledger (`isVotingLedger()`, seq ≡ 255), it attaches its fee votes and the amendments it votes for to that validation, along with its server version. One round later, when `prev_ledger` is the flag ledger, `onClose` collects the trusted validations of the voting ledger (`prev_ledger`'s parent), requires at least `quorum` of them, and tallies them into the fee and amendment pseudo-transactions for `flag + 1`.

The negative UNL does not use votes carried in validations. `NegativeUNLVote::doVoting` emits the `ttUNL_MODIFY` pseudo-transactions directly. What the negative UNL is, how validators are scored and picked, and how it changes the quorum are covered in [unl.md](unl.md).

## C++ reference

| Pseudocode / concept | C++ |
|---|---|
| Flag-ledger interval, 256 | `kFlagLedgerInterval` |
| Flag-ledger / voting-ledger predicate | `Ledger::isFlagLedger`, `Ledger::isVotingLedger` |
| Add vote pseudo-transactions to the proposed set | `RCLConsensus::Adaptor::onClose` |
| Collect the voting ledger's trusted validations | `RCLValidations::getTrustedForLedger` |
| Attach fee / amendment votes to a validation | `RCLConsensus::Adaptor::validate`, `FeeVote::doValidation`, `AmendmentTable::doValidation` |
| Tally fee / amendment votes | `FeeVoteImpl::doVoting`, `AmendmentTable::doVoting` |
| Score validator reliability, emit `ttUNL_MODIFY` | `NegativeUNLVote::doVoting`, `NegativeUNLVote::buildScoreTable` |
| Apply the pseudo-transactions | `Change::applyFee`, `Change::applyAmendment`, `Change::applyUNLModify` |
| Enact pending negative UNL on the flag-ledger build | `Ledger::updateNegativeUNL`, `buildLedgerImpl` |
