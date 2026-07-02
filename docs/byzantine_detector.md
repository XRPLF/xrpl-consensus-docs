# Byzantine Behavior Detector

The Byzantine behavior detector spots a validator signing more than one validation for the same ledger sequence, the on-network sign of equivocation. It is purely diagnostic. It changes no consensus state: a flagged validation is only logged. It is the peer-facing counterpart of the node's own [anti-equivocation floor](04_phase_accepted.md#broadcasting-the-validation), which stops *this* node from ever signing two validations at one height.

Detection happens when a validation is recorded. `Validations::add` keeps, per validator, the highest sequence it has seen ([Incoming validations](04_phase_accepted.md#incoming-validations)). When a new validation does not advance that sequence, `add` looks at the one already tracked for that sequence and returns a status:

- **`Conflicting`**: same sequence, but a different ledger, or the same sequence and ledger signed at a different time. Two distinct signatures at one height, the clearest equivocation signal.
- **`Multiple`**: same sequence, ledger, and sign time, but a different cookie. Usually one operator running two instances with the same key rather than a malicious fork.
- **`BadSeq`**: a sequence not greater than one already seen from that validator, without matching either case above. Out of order, but not a same-height conflict.

A validation with any of these statuses does not enter the store used for quorum and fork choice. `add` returns before recording it there. Only `Conflicting` and `Multiple` are logged as Byzantine behavior. `BadSeq` and `Stale` (a validation too old to be current) are dropped without a Byzantine note.

The log line is written in `handleNewValidation`, on the XRP Ledger side, tagged `Byzantine Behavior Detector` and naming the validator and the sequence. A misbehaving **trusted** validator logs at `error`, an untrusted one at `info`. The node still relays a flagged validation rather than dropping it, so peers observe the same misbehavior and their operators take independent notice.

Example log lines (the journal partition is `Validations`, and the validator key, sequence, and ledger hash are shortened here):

```
Validations:ERR Byzantine Behavior Detector: trusted n9JqFzp...282Pz: Multiple validations for 361188/0817E993...FACF!
Validations:ERR Byzantine Behavior Detector: trusted n9JqFzp...282Pz: Conflicting validation for 361188!
```

## C++ reference

| Concept | C++ |
|---|---|
| Classify a recorded validation | `Validations::add` -> `ValStatus` (`Validations.h`) |
| Status values | `ValStatus::{Current, Stale, BadSeq, Multiple, Conflicting}` |
| Per-validator sequence guard | `Validations::seqEnforcers_` (`SeqEnforcer<Seq>`) |
| Per-sequence index the check reads | `Validations::bySequence_` |
| Logging and relay decision | `handleNewValidation` (`RCLValidations.cpp`). Journal `"Byzantine Behavior Detector"`, `error` if trusted else `info` |
