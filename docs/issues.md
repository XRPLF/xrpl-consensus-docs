# Consensus Issues

Bugs and issues in the `xrpld` consensus code, found by reading the source and confirmed against it. Each entry cites `file:line` and, where one exists, a reproducing unit test in `src/test/consensus/`. Findings are tiered by how certain and how serious they are. Line numbers are as of the reviewed tree and may drift.

## Finality not locally enforced (design gap)

### `checkAccept` advances the fully validated ledger across a conflict, with no ancestry guard

This is about the **fully validated ledger** (`setValidLedger`, reached only through `checkAccept`), the finality pointer the docs describe as never rolled back. It is not about the preferred ledger (`getPreferred`), which selects what the next ledger is built on and is expected to switch on any minor divergence. Moving the preferred ledger is routine, moving the fully validated pointer across a conflict is not.

`LedgerMaster::checkAccept(ledger)` (`LedgerMaster.cpp:944`) advances the fully validated ledger when a candidate is newer than the current one, passes `canBeCurrent`, and holds a quorum of trusted full validations. It never checks that the candidate descends from the ledger the node has already fully validated. `canBeCurrent` (`LedgerMaster.cpp:353`) only bounds the sequence and the close-time window, so it does not close the gap. The node guards two neighboring actions with `LedgerMaster::isCompatible`, issuing its own validation (`RCLConsensus.cpp`) and switching its last closed ledger (`NetworkOPs.cpp`), but the fully-validated advance has no equivalent guard.

Reproduction: `CheckAcceptFork_test`. It fully validates `L2` (seq 3), confirms `isCompatible` reports the sibling-descended `L3'` (seq 4, on `L2'`) incompatible, then feeds `L3'` a quorum of trusted validations. `checkAccept(L3')` moves `getValidLedgerIndex` to `L3'`, so the finalized seq-3 ledger is retroactively `L2'` rather than `L2`. The simulation model does not do this: `csf`'s `Peer::checkFullyValidated` has an `isAncestor` guard the production `LedgerMaster` lacks, and `ForkSafetySim_test` (driving the model) shows the honest node keeping `L2`.

Whether this is a defect is a genuine design question, not a clear bug. It fires only once a quorum of trusted validators has fully validated the conflicting branch. Under the UNL-overlap and sub-20%-Byzantine assumption two conflicting ledgers cannot both reach quorum, so by the time this triggers the network-level safety assumption is already violated, and no per-node check can restore safety. Given a real fork, following the newer quorum of one's own trusted validators is a defensible way to rejoin the majority rather than strand the node. The narrow, objective finding is that production does not locally enforce the irreversibility the docs state and the model enforces: it rolls the finality pointer across a conflict silently, rather than detecting the finalized conflict and halting or alarming. The docs' "never rolled back" wording is therefore only true while the network assumption holds, not unconditionally.

## Latent inconsistency

### `consensusBuilt` requires strictly more than quorum where every other path requires at least quorum

The opportunistic scan in `LedgerMaster::consensusBuilt` selects an alternate ledger to fully validate with `valCount > neededValidations` (`LedgerMaster.cpp:1157`), a strict comparison. Every other quorum test uses at-least: `checkAccept(hash, seq)` at `LedgerMaster.cpp:897`, and `checkAccept(ledger)` returns early only when `tvc < minVal` at `LedgerMaster.cpp:962`. `neededValidations` is `quorum()`.

So a competing ledger sitting at exactly quorum is skipped by the post-build scan, one validation short of the threshold the rest of the code uses. The continuous `>=` path (`handleNewValidation` to `checkAccept(hash, seq)`) still adopts it when the next validation for it arrives, so the observable effect is at most a one-event delay, not a permanent miss. The inconsistency itself is the issue.

## Diagnostic and narrow-window limitations

### Equivocation detection degrades after five minutes, and the per-validator floor resets after ten

`Validations::add` (`Validations.h`) classifies a second validation at a sequence already seen. Within `validationCurrentWall` (about five minutes) two validations at one sequence for different ledgers return `Conflicting`. Past that window the stored entry is replaced by the newer validation, and the equivocation check then compares the new validation against itself, so the same double-sign returns `BadSeq` rather than `Conflicting`. The Byzantine behavior detector logs only `Conflicting` and `Multiple`, so an equivocation split by more than five minutes is not flagged.

Separately, the per-validator `SeqEnforcer` (`Validations.h`) resets its floor to zero after `validationSetExpires` (about ten minutes) with no newer sequence recorded, after which a validation at or below a previously seen sequence is accepted as `Current`.

Reproduction: `ValidationSequenceTiming_test` walks all three windows (under five minutes, five to ten, past ten) for a lower sequence and for same-sequence equivocation.

Impact is limited. The Byzantine detector is diagnostic only and changes no consensus state, so the degraded classification costs a log line, not safety. The floor reset is mitigated for a running validator, which validates every few seconds and so never idles ten minutes, and for a restart by `maxDisallowedLedger`, the on-disk floor that survives reboot.

## Known edge cases (characterized, not novel)

### Close-time rounding is not a fixed point

`effCloseTime` over `roundCloseTime` (`LedgerTiming.h`) can map a close time the peers agreed on to a value that, rounded again, lands in a different resolution bin, because the clamp to `parentCloseTime + 1s` can push the result to the next multiple of the resolution. Two nodes can therefore record different close times for a ledger they believe they agreed on.

This is characterized by `xrpld`'s own `Consensus_test::testConsensusCloseTimeRounding` (`Consensus_test.cpp:612`), which engineers the discrepancy deliberately and notes in a comment that the consensus timing is what prevents it in practice. `asCloseTime` (`Consensus.h`) also rounds without the `parentCloseTime + 1s` clamp, so a vote can name a time below the parent close, but since every node rounds identically and the clamp is applied once at build, this does not cause divergence. Recorded here as a known subtlety, not a new bug.

### The 256-ledger skip-list horizon splits a single chain

`RCLValidatedLedger` derives ancestry from the on-ledger skip list, which carries the prior 256 hashes. Two ledgers on the same chain more than 256 sequences apart are treated as unrelated by the `LedgerTrie`, and the read path imposes no size cap on the skip list (`RCLValidations.h`). This is intentional and documented in `RCLValidations.h`, and the ledger hash commits the skip list, so an oversized or forged one changes the ledger hash and fails validation. `TrieSkipList_test` explores the boundary (256 apart stays one branch, 257 splits) and the oversized-list read. Not a bug, listed so the horizon is on the record.

### Negative-UNL cap rounds up

`NegativeUNLVote::findAllCandidates` caps the list at `ceil(unl.size() * 0.25)` (`NegativeUNLVote.cpp:244`). For a UNL size that is not a multiple of four this rounds above 25% (for example 51 of 201, 25.4%). The effect on quorum is bounded by the independent 60%-of-original floor in the quorum formula, so this is a marginal spec-versus-code nuance rather than a safety issue.

## Investigated and found not to be issues

Recorded so they are not re-investigated.

- **Proposer versus observer asymmetry** in dispute voting (`DisputedTx::updateVote`, the non-proposing branch uses a simple majority) and in the close-time tally (`Consensus.h:1534`, only a proposing node counts itself). This is intentional: an observing node is not authoritative, it emits only partial validations that do not count toward quorum, and two proposing nodes with the same peer positions use identical formulas. The observer branch is commented "don't let us outweigh a proposing node, just recognize consensus".
- **Fork-choice divergence in `LedgerTrie::getPreferred`.** The margin-over-uncommitted test and the id-based tie-break are deterministic functions of support and ledger id, identical across honest nodes with the same validation set.
- **Negative-UNL validator selection** (`NegativeUNLVote::choose`) is an order-independent minimum of `id ^ randomPad`, so `hash_map` iteration order does not change the chosen validator across nodes.
- **`checkConsensus` MovedOn ratio** (`Consensus.cpp`) can exceed 100% because its denominator is the current proposer count. `MovedOn` means the round is abandoned and the node resyncs, which is benign, so an eager trigger is not a safety problem.
- **Immediate tick when more than half the previous proposers have already sent positions** (`Consensus.h`) followed by the majority-closed branch of `shouldCloseLedger` (`Consensus.cpp`). Closing to catch up when a majority has already closed is the intended behavior.
