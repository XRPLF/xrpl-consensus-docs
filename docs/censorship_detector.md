# Censorship Detector

The censorship detector tracks transactions in the set this node proposed that keep being left out of accepted ledgers, and logs a warning when one has been excluded for too many ledgers. It is purely diagnostic. It reads no consensus state and changes none: positions, votes, validations, and the built ledger are decided elsewhere, and the detector's only output is a log line. It is specific to the XRP Ledger and lives in the RCL adaptor (`RCLConsensus::Adaptor`), not the generic consensus engine, which has no notion of it.

The detector holds one entry per tracked transaction, a `(txid, seq)` pair. `txid` identifies the transaction. `seq` is the sequence of the ledger that was being built when the transaction first entered the proposed set, preserved across every later round the transaction has stayed in that set. The entries are exactly the transactions in the current proposed set that earlier rounds left out of the accepted ledger.

Re-registering a tracked transaction keeps its original `seq`, so `seq` records when tracking began, not the latest round. A transaction dropped from the proposed set is removed from the tracker. If it returns in a later round, it is tracked afresh from the current sequence.

The detector runs from adaptor callbacks:

- **`propose`**, from [`on_close`](03_phase_establish.md#taking-our-position) when the node takes its position and holds the correct last closed ledger. It registers the proposed set: every transaction in the set about to be proposed, each paired with the sequence of the ledger being built. Transactions carried over from a previous round keep their earlier `seq`. Transactions no longer proposed are dropped.
- **`check`**, from [`do_accept`](04_phase_accepted.md#updating-the-censorship-detector) after the ledger is built, only when the node holds the correct last closed ledger and the round ended in the `Yes` consensus state. It reconciles the tracker against the accepted set. On a round that ended any other way, `check` does not run and the tracker carries over unchanged.
- **`reset`**, from `on_mode_change`, when the consensus mode changes away from `PROPOSING` or `OBSERVING`, as on a bow-out (see [Reaching consensus](03_phase_establish.md#reaching-consensus)) or a wrong-ledger switch (see [Phase Dispatch](01_phase_dispatch.md#timer-entry)). It clears the tracker, so a node that has stopped participating does not warn on stale entries.

`check` removes the entries it can account for: transactions that made it into the accepted set, and transactions the build could not include for a known reason (they failed to decode or apply, or were held over to retry next round). What remains is the transactions that were eligible yet left out. For each, `check` computes the wait as the current ledger sequence minus the entry's `seq`. When that wait is a nonzero multiple of `kCensorshipWarnInternal` (15 ledgers), it logs a warning naming the transaction, the ledger tracking began on, and the current ledger. The warning is informational. The transaction stays in the tracker and continues to be proposed.

Example log line (the journal partition is `CensorshipDetector`, and the transaction id is shortened here):

```
CensorshipDetector:WRN Potential Censorship: Eligible tx 5B2C...0F19, which we are tracking since ledger 360500 has not been included as of ledger 360515.
```

## C++ reference

| Concept | C++ |
|---|---|
| Detector class | `RCLCensorshipDetector` (`RCLCensorshipDetector.h`) |
| Detector instance | `RCLConsensus::Adaptor::censorshipDetector_` |
| Tracker entry `(txid, seq)` | `RCLCensorshipDetector::TxIDSeq` |
| Register the proposed set | `RCLCensorshipDetector::propose`, from `RCLConsensus::Adaptor::onClose` |
| Reconcile and warn | `RCLCensorshipDetector::check`, from `RCLConsensus::Adaptor::doAccept` |
| Clear the tracker | `RCLCensorshipDetector::reset`, from `RCLConsensus::Adaptor::onModeChange` |
| Seq-preserving and removal set passes | `generalizedSetIntersection`, `removeIfIntersectOrMatch` (`algorithm.h`) |
| Warn interval, 15 ledgers | `RCLConsensus::kCensorshipWarnInternal` |
| Warning log | journal `"CensorshipDetector"`, level warn |
