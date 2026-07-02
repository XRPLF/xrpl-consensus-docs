# Code Organization

## Overview

Consensus in `xrpld` is a generic algorithm wrapped in the XRP Ledger code that drives it. The generic algorithm decides consensus over abstract types, a transaction set known only by its hash, a position, a ledger id. It never inspects a real transaction or builds a ledger, so in principle it could run over a different ledger. Everything XRP Ledger-specific is supplied from outside.

The code connected to consensus splits across these areas, each covered in a section below:

* **The generic algorithm** (`src/xrpld/consensus/`) is the generic core described above.
* **The RCL binding** (`src/xrpld/app/consensus/`) plugs the XRP Ledger into the generic algorithm. `RCLConsensus::Adaptor` and the `RCLCx*` wrapper types supply the concrete types and the ledger-specific work the generic code calls out for: building real transaction sets, fee/amendment/negative-UNL voting, and building ledgers.
* **The driver** (`NetworkOPs`, `src/xrpld/app/misc/`) runs consensus on a timer, sets the node's operating mode, and routes incoming validations.
* **The overlay** (`src/xrpld/overlay/`) is the peer-to-peer layer that carries proposals, validations, and status between peers.
* **Ledger state and full validation** (`src/xrpld/app/ledger/`) supply the prior ledgers consensus builds on, build the agreed ledger, and advance the fully validated ledger. `LedgerMaster` is the center of it.
* **Trust and voting** (`src/xrpld/app/misc/`) supply the trusted validator set and quorum (`ValidatorList`) and cast the per-round fee, amendment, and negative-UNL votes.

One boundary is exact: the generic code is precisely what lives under `src/xrpld/consensus/` - everything else is application code. Beyond that, "the driver" and the subsystem groupings below are this doc's labels, not layers the code declares.

## The generic algorithm

`src/xrpld/consensus/` holds the consensus algorithm as two `Adaptor`-parameterized class templates. `Consensus<Adaptor>` runs a round through the `OPEN`, `ESTABLISH`, and `ACCEPTED` phases and converges trusted validators on one transaction set and close time. `Validations<Adaptor>` stores validators' validations and answers the fork choice, which prior ledger to build on ([preferred_ledger.md](preferred_ledger.md)). The rest of the directory is the data types they work over, the tunables, the dispute object, and the trie.

### `Consensus<Adaptor>` (`Consensus.h`)

`Consensus<Adaptor>` is the consensus state machine. One instance runs one round at a time, through the `OPEN`, `ESTABLISH`, and `ACCEPTED` phases. It is not thread-safe - the caller serializes access (the RCL wrapper does this under a lock). It keeps no clock of its own: the entry points that advance a round take the current network-adjusted time as a `now` argument and store it in `now_`. The read-only accessors take no `now`.

| Entry point | Role |
|---|---|
| `startRound` | begin a round: acquire the correct prior ledger if handed the wrong one, then `startRoundInternal` ([Starting the round](05_start_consensus_round_again.md#starting-the-round)) |
| `timerEntry` | the periodic tick: refresh `now_`, run `checkLedger`, dispatch to `phaseOpen` or `phaseEstablish` ([Phase Dispatch](01_phase_dispatch.md)) |
| `peerProposal` | a peer's position arrived: buffer it in `recentPeerPositions_`, then `peerProposalInternal` ([Receiving a peer's position](03_phase_establish.md#receiving-a-peers-position)) |
| `gotTxSet` | a fetched transaction set arrived: record peers' votes on it ([Receiving a peer's position](03_phase_establish.md#receiving-a-peers-position)) |
| `simulate` | standalone-only forced accept (the `ledger_accept` RPC): calls `onForceAccept` |

The entry points delegate to private methods that carry the phase logic:

* `checkLedger` confirms the round is still on the network's preferred prior ledger. If it is not, it calls `handleWrongLedger` to switch ([Phase Dispatch](01_phase_dispatch.md), fork choice in [preferred_ledger.md](preferred_ledger.md)).
* `playbackProposals` replays peer positions buffered before the round started, both when a round begins normally (`startRoundInternal`, [Resetting per-round state](05_start_consensus_round_again.md#resetting-per-round-state)) and after a wrong-ledger switch (`handleWrongLedger`, [Phase Dispatch](01_phase_dispatch.md)).
* `phaseOpen` decides whether to close the open ledger ([OPEN Phase](02_phase_open.md)).
* `closeLedger` takes our position and enters the `ESTABLISH` phase ([Closing the open ledger](03_phase_establish.md#closing-the-open-ledger)).
* `phaseEstablish` runs each `ESTABLISH` tick ([The ESTABLISH tick](03_phase_establish.md#the-establish-tick)), calling:
  * `updateOurPositions` to revise our position: the close-time tally, the dispute loop, the stale-peer drop, and the re-broadcast all live inside it.
  * `shouldPause` to brake for laggards ([Pausing for laggards](03_phase_establish.md#pausing-for-laggards)).
  * `haveConsensus` to decide whether the round can accept ([Reaching consensus](03_phase_establish.md#reaching-consensus)).
* `createDisputes` / `updateDisputes` build and refresh the per-transaction disputes.
* `leaveConsensus` is our bow-out, on expiry or a wrong-ledger switch.
* `asCloseTime` rounds a raw close time to the round's bin (it calls `roundCloseTime`).

On the `ESTABLISH`-to-`ACCEPTED` transition `phaseEstablish` hands the round's `result_` to the adaptor's `onAccept`, which builds and validates the ledger ([ACCEPTED Phase](04_phase_accepted.md)). The inner `MonitoredMode` holds `mode_` and calls the adaptor's `onModeChange` on every mode change.

Its members split into the current round's working state and the values carried across rounds.

The current-round state is cleared or rebuilt when a round starts (`startRoundInternal`, [Resetting per-round state](05_start_consensus_round_again.md#resetting-per-round-state) or `closeLedger`, [Closing the open ledger](03_phase_establish.md#closing-the-open-ledger), for the `ESTABLISH` counters):

| Member | Holds |
|---|---|
| `phase_` | the current `ConsensusPhase` |
| `mode_` | the current `ConsensusMode` (wrapped in `MonitoredMode`) |
| `now_` | the network-adjusted time |
| `prevLedgerID_` | id of the prior ledger this round builds on |
| `previousLedger_` | the prior ledger itself |
| `closeResolution_` | this round's close-time bin size |
| `openTime_` | stopwatch for the `OPEN` phase |
| `result_` | our stance this round (`ConsensusResult`): unset during `OPEN`, set through `ESTABLISH` and `ACCEPTED` |
| `acquired_` | transaction sets we currently hold, by hash |
| `currPeerPositions_` | trusted peers' current positions this round |
| `rawCloseTimes_` | our and peers' unrounded close times |
| `deadNodes_` | peers that have bowed out this round |
| `closeTimeAvalancheState_` | the close-time vote's avalanche level |
| `haveCloseTimeConsensus_` | whether the close time has settled |
| `convergePercent_` | round duration so far as a percent of the previous round's |
| `peerUnchangedCounter_` | `ESTABLISH` ticks since any peer last changed a dispute vote |
| `establishCounter_` | total `ESTABLISH` ticks this round |

The rest carries across rounds:

| Member | Holds |
|---|---|
| `firstRound_` | whether this is the instance's first round |
| `clock_` | the steady (monotonic) clock (an injected dependency) |
| `prevCloseTime_` | our own close time of the prior ledger |
| `prevProposers_` | proposer count of the previous round |
| `prevRoundTime_` | duration of the previous round |
| `recentPeerPositions_` | recently received positions, replayed into a new round |

Two close-decision routines are free functions, declared in `Consensus.h` and defined in `Consensus.cpp`. 
* `shouldCloseLedger` is the `OPEN`-phase close test. 
* `checkConsensus` is the `ESTABLISH`-phase outcome test. Its helper `checkConsensusReached` is the supermajority test, and `participantsNeeded` converts a percentage to a vote count.

`Consensus` calls the `Adaptor` template parameter for everything application-specific: acquiring ledgers and transaction sets, taking and broadcasting our position, building and validating the ledger, and the inputs to its decisions. The concrete callbacks are in [The RCL binding](#the-rcl-binding).

### Data types (`ConsensusTypes.h`, `ConsensusProposal.h`, `DisputedTx.h`)

`ConsensusTypes.h` defines the round's enums, [`ConsensusMode`, `ConsensusPhase`, and `ConsensusState`](terminology.md), along with `ConsensusTimer` (phase durations) and `ConsensusCloseTimes` (the round's raw close times). `ConsensusResult<Adaptor>` aggregates the round's stance. It is the `result_` member, passed to `onAccept` ([Round result](03_phase_establish.md#round-result)).

`ConsensusProposal` (`ConsensusProposal.h`) is one position: the prior ledger it builds on, the proposed set's hash, the close time, and a [sequence number that tracks revisions and bow-out](03_phase_establish.md#position). This is the generic `Position`. A trusted peer's arrives wrapped as `RCLCxPeerPos`.

`DisputedTx` (`DisputedTx.h`) is one not-yet-unanimous transaction with our vote and each peer's. The voting it drives is in [Voting on disputes](03_phase_establish.md#voting-on-disputes) and [Reaching consensus](03_phase_establish.md#reaching-consensus).

### Tunables (`ConsensusParms.h`)

`ConsensusParms` is the bundle of protocol tuning constants (timings plus the consensus percentages and round counts), reached through `adaptor_.parms()`. The avalanche helper `getNeededWeight` (a free function) reads the `avalancheCutoffs` map of `AvalancheState` levels and returns the yes-share for the caller's level ([The avalanche](03_phase_establish.md#the-avalanche)).

| Parameter | Controls |
|---|---|
| `ledgerIdleInterval` | idle time before closing an empty ledger |
| `ledgerMinClose` | minimum `OPEN` time before closing |
| `ledgerMinConsensus` | minimum `ESTABLISH` time before deciding |
| `ledgerMaxConsensus` | when a lone node may accept, and the pause-for-laggards cap |
| `ledgerAbandonConsensusFactor`, `ledgerAbandonConsensus` | the expiry bound |
| `avMinConsensusTime` | floor for the previous-round duration in converge percent |
| `avalancheCutoffs`, `avMinRounds` | the avalanche ladder and its per-level tick gate |
| `avCtConsensusPct` | close-time settle threshold (75%) |
| `avStalledRounds` | ticks of no change before a dispute counts as stalled |
| `minConsensusPct` | transaction-set supermajority (80%) |
| `proposeINTERVAL`, `proposeFRESHNESS` | how often we re-broadcast, how long a peer position stays fresh |

`kMaxPausePhase` is not in `ConsensusParms`. It is a local constant inside `Consensus::shouldPause`.

### Validations and the trie (`Validations.h`, `LedgerTrie.h`)

`Validations<Adaptor>` (`Validations.h`) stores the current and recent validations from listed nodes, indexed by node, by ledger id, and by sequence. It maintains a `LedgerTrie` for the fork choice. In the XRP Ledger build it is instantiated as `RCLValidations` (= `Validations<RCLValidationsAdaptor>`).

| Method | Role |
|---|---|
| `add` | record a validation ([Incoming validations](04_phase_accepted.md#incoming-validations)). Returns a `ValStatus` (`Current`, `Stale`, `BadSeq`, `Multiple`, `Conflicting`) that drives [Byzantine detection](byzantine_detector.md) |
| `getPreferred` / `getPreferredLCL` | the preferred-ledger fork choice ([preferred_ledger.md](preferred_ledger.md)). `getPreferredLCL` adds the peer-count fallback |
| `getNodesAfter` | how many trusted validators built on a ledger's descendant. Backs the adaptor's `proposersFinished` ([Reaching consensus](03_phase_establish.md#reaching-consensus)) |
| `getTrustedForLedger` | the trusted full validations for a given (id, seq) (a list, the count is `numTrustedForLedger`). Read by the quorum tally ([Advancing the fully validated ledger](04_phase_accepted.md#advancing-the-fully-validated-ledger)) and negative-UNL scoring ([unl.md](unl.md)) |
| `numTrustedForLedger` | trusted full validation count for a ledger. Backs the adaptor's `proposersValidated` ([OPEN Phase](02_phase_open.md)) |
| `canValidateSeq` | the in-memory anti-equivocation gate ([Broadcasting the validation](04_phase_accepted.md#broadcasting-the-validation)) |
| `trustChanged` | when the [trusted set](unl.md) changes, add or drop those validators' support in the [trie](preferred_ledger.md) |

`ValidationParms` (in this file) holds the validation timing constants: the laggard cutoff ([Pausing for laggards](03_phase_establish.md#pausing-for-laggards)), the stored-set retention and anti-equivocation idle reset ([Broadcasting the validation](04_phase_accepted.md#broadcasting-the-validation)), and the currency windows `isCurrent` uses. `SeqEnforcer<Seq>` is the test-and-set that enforces increasing validation sequences, behind both `canValidateSeq` (our own [anti-equivocation floor](04_phase_accepted.md#broadcasting-the-validation)) and the [Byzantine detector](byzantine_detector.md).

`LedgerTrie<Ledger>` (`LedgerTrie.h`) is the data structure behind the fork choice. It maps each ledger's ancestry into a trie of `Span`s and answers `getPreferred`: the deepest ledger whose lead survives every uncommitted validator defecting. The full walk is specified in [preferred_ledger.md](preferred_ledger.md).

## The RCL binding

`src/xrpld/app/consensus/` binds the generic code to the XRP Ledger, filling in each generic type with a concrete one:

| Generic | RCL binding | Wraps |
|---|---|---|
| `Consensus<Adaptor>` | `RCLConsensus::Adaptor` | the app: `LedgerMaster`, voting, messaging |
| `Adaptor::Ledger_t` | `RCLCxLedger` | `shared_ptr<Ledger const>` |
| `Adaptor::TxSet_t` | `RCLTxSet` | `SHAMap` (a tx is `RCLCxTx` over a `SHAMapItem`) |
| `Adaptor::PeerPosition_t` | `RCLCxPeerPos` | a signed `ConsensusProposal` |
| `Adaptor::NodeID_t` | `NodeID` | a peer's identifier |
| `Validations<Adaptor>` | `RCLValidations` (= `Validations<RCLValidationsAdaptor>`) | the validation store |
| `Validations` `Validation` | `RCLValidation` | `STValidation` |
| `LedgerTrie` `Ledger` | `RCLValidatedLedger` | a ledger's id, seq, and skip-list ancestry |

### `RCLConsensus` and `RCLConsensus::Adaptor` (`RCLConsensus.h`, `RCLConsensus.cpp`)

`RCLConsensus` owns the `Consensus` instance and serializes access under a recursive mutex: its public methods (`timerEntry`, `peerProposal`, `gotTxSet`, `startRound`, `simulate`, `prevLedgerID`) take the lock and forward. `NetworkOPs` reads `validating`, `prevProposers`, `prevRoundTime`, and `mode` from atomic mirrors without the lock.

`RCLConsensus::Adaptor` is the concrete `Adaptor`. Its members implement the generic callbacks, called only by `Consensus` under the lock, and tie consensus to the app (`LedgerMaster`, the fee and negative-UNL vote casters, the censorship detector, the validator keys). The accept job is the exception: it runs without the lock while the round stays frozen in the `ACCEPTED` phase, then starts the next round ([ACCEPTED Phase](04_phase_accepted.md), [Starting the Next Round](05_start_consensus_round_again.md)).

The adaptor splits into the generic callbacks and the round-lifecycle work:

| Adaptor member | Role |
|---|---|
| `acquireLedger`, `acquireTxSet` | fetch a ledger or transaction set, locally or from peers ([Acquisition](#acquisition)) |
| `hasOpenTransactions` | whether the open ledger holds any transactions |
| `proposersValidated`, `proposersFinished` | trusted-validator counts for the close ([OPEN Phase](02_phase_open.md)) and consensus ([Reaching consensus](03_phase_establish.md#reaching-consensus)) tests |
| `getPrevLedger` | the network's preferred prior ledger (calls `Validations::getPreferred`, see [preferred_ledger.md](preferred_ledger.md)) |
| `onClose` | take our position: build the set (with flag-ledger votes) and opening proposal ([Taking our position](03_phase_establish.md#taking-our-position)) |
| `onAccept` / `onForceAccept` | dispatch the accept job (or run it inline for `simulate`) |
| `doAccept` | the accept job body ([ACCEPTED Phase](04_phase_accepted.md)) |
| `buildLCL` | build the new ledger from the agreed set (calls `buildLedger`, see [Building the ledger](04_phase_accepted.md#building-the-ledger)) |
| `validate` | sign and broadcast our validation, attaching fee/amendment votes when it validates a voting ledger ([Broadcasting the validation](04_phase_accepted.md#broadcasting-the-validation)) |
| `notify` | broadcast a `TMStatusChange` status message about the ledger we hold ([Announcing the ledger](04_phase_accepted.md#announcing-the-ledger)) |
| `propose`, `share` | broadcast our position, and relay a peer position or transaction over the overlay and hand a transaction set to `InboundTransactions` |
| `preStartRound` | recompute `validating_` and decide whether we propose ([Deciding whether to propose](05_start_consensus_round_again.md#deciding-whether-to-propose)) |
| `updateOperatingMode` | drop out of `FULL` mode when no peer has proposed any position |
| `onModeChange` | record a consensus-mode change: update the atomic mode mirror, and reset the censorship detector when the node stops proposing or observing |
| `parms`, `validating` | the consensus tuning parameters and whether this node signs validations, both read by the engine |
| `getValidLedgerIndex`, `getQuorumKeys`, `laggards`, `validator`, `haveValidated` | inputs to `shouldPause` ([Pausing for laggards](03_phase_establish.md#pausing-for-laggards)) |

### The `RCLCx*` types

The concrete types the template parameters resolve to:

* `RCLCxLedger` wraps `shared_ptr<Ledger const>` and exposes the seq, id, parent id, close time, and close-time resolution `Consensus` needs.
* `RCLTxSet` wraps a `SHAMap` (a tx is `RCLCxTx` over a `SHAMapItem`) and supplies the `compare` tree diff that finds disputes.
* `RCLCxPeerPos` is a signed `ConsensusProposal`.
* `RCLCensorshipDetector` tracks transactions repeatedly left out of accepted ledgers (diagnostic only, see [censorship_detector.md](censorship_detector.md)).

### The RCL validation types

* `RCLValidation` wraps an `STValidation`.
* `RCLValidatedLedger` wraps a ledger for the `LedgerTrie`, taking ancestry from the skip list (ledgers more than 256 apart count as unrelated).
* `RCLValidations` is the alias `Validations<RCLValidationsAdaptor>`.
* `handleNewValidation` (a free function) is the single intake path: it sets trust from the current UNL, stores the validation through `Validations::add` ([Incoming validations](04_phase_accepted.md#incoming-validations), [Byzantine detector](byzantine_detector.md)), and on a new trusted validation calls `LedgerMaster::checkAccept` to advance the fully validated ledger ([Advancing the fully validated ledger](04_phase_accepted.md#advancing-the-fully-validated-ledger)).

## NetworkOPs (the driver)

`NetworkOPsImp` (`src/xrpld/app/misc/NetworkOPs.cpp`) does the driving through these entry points:

* `processHeartbeatTimer` (a `JtNetopTimer` job) is the heartbeat: it sets the operating mode and, with enough peers, calls `RCLConsensus::timerEntry` with the network-adjusted `now` ([Phase Dispatch](01_phase_dispatch.md)).
* `endConsensus` then `beginConsensus` run the round lifecycle: the accept job's second half settles sync state, rebuilds the trusted set and quorum, and starts the next round ([Ending the round](05_start_consensus_round_again.md#ending-the-round), [Beginning consensus](05_start_consensus_round_again.md#beginning-consensus)).
* `recvValidation` takes in peer validations, forwarding each to `handleNewValidation` and deciding whether to relay it ([Incoming validations](04_phase_accepted.md#incoming-validations)).

## The overlay

The pieces of `src/xrpld/overlay/` that consensus touches:

* `Overlay` (`Overlay.h`) exposes `size` (the peer count the heartbeat checks), `broadcast` (send our proposal or validation), and `checkTracking` (mark diverged peers).
* `PeerImp::onMessage` (`detail/PeerImp.h`) is the receive side, dispatching each message to its consensus handler.
* `Peer::cycleStatus` clears a peer's stale closed-ledger report.
* `HashRouter` (`include/xrpl/core/HashRouter.h`) deduplicates relayed messages. `addSuppression` pre-registers a hash so our own echoed validation is ignored.

The consensus-related messages, all defined in `include/xrpl/proto/xrpl.proto` (each with an `mt…` wire tag):

| Message / tag | Carries | Used for |
|---|---|---|
| `TMProposeSet` / `mtPROPOSE_LEDGER` | A signed position: proposer key, propose sequence, proposed tx-set hash, close time, prior-ledger hash, and an added/removed-transaction delta | A peer's consensus position. Intake is `peerProposal` ([Receiving a peer's position](03_phase_establish.md#receiving-a-peers-position)) |
| `TMValidation` / `mtVALIDATION` | A serialized `STValidation` | A peer's validation of a ledger. Intake is `handleNewValidation` ([Incoming validations](04_phase_accepted.md#incoming-validations)) |
| `TMStatusChange` / `mtSTATUS_CHANGE` | Node status and event, the latest closed ledger (seq, hash, parent hash), network time, and the ledger range the node can serve | Announcing the ledger we hold ([Announcing the ledger](04_phase_accepted.md#announcing-the-ledger)). Feeds peer tracking and the peer-count fork fallback |
| `TMTransaction` / `mtTRANSACTION` | One serialized transaction | Relaying a transaction into peers' open ledgers |
| `TMHaveTransactions` / `TMTransactions` (`mtHAVE_TRANSACTIONS` / `mtTRANSACTIONS`) | Advertise, then transfer, a batch of transactions by hash | Bulk transaction propagation |
| `TMHaveTransactionSet` / `mtHAVE_SET` | A transaction-set hash and the sender's status for it: has it, can get it from a peer, or needs it (`tsHAVE` / `tsCAN_GET` / `tsNEED`) | Locating a peer that holds a proposed transaction set |
| `TMGetLedger` / `TMLedgerData` (`mtGET_LEDGER` / `mtLEDGER_DATA`) | Request and reply for a ledger or a candidate transaction set (`itype` selects which, and the hash may be a tx-set hash) | Acquiring the tx sets proposals name and the ledgers validations reference (`InboundLedgers`, `InboundTransactions`) |
| `TMGetObjectByHash` / `mtGET_OBJECTS` | Request and reply for SHAMap objects by hash (ledger, transaction, and state nodes) | Fetching ledger contents during acquisition |

The first three are consensus proper: the position, validation, and status messages the algorithm itself sends and receives. The rest move the data those reference but consensus does not decide on, the transactions a proposal names and the ledgers a validation points to, fetched from peers on demand.

## Ledger state and full validation (`src/xrpld/app/ledger/`)

`LedgerMaster` (`LedgerMaster.h`) owns the node's view of closed and validated ledgers:

| Method | Role |
|---|---|
| `switchLCL` | adopt the built ledger as our last closed ledger ([Switching the last closed ledger](04_phase_accepted.md#switching-the-last-closed-ledger)) |
| `consensusBuilt` | the accept-time advance of the fully validated ledger |
| `checkAccept` (two overloads) | fully validate a ledger if a quorum backs it ([Advancing the fully validated ledger](04_phase_accepted.md#advancing-the-fully-validated-ledger)) |
| `setValidLedger` | adopt a ledger as fully validated and run the downstream effects |
| `tryAdvance` -> `doAdvance` | wake the background advance that publishes validated ledgers and fetches missing ones |
| `isCompatible` | confirm a built ledger does not conflict with the validated chain ([Checking the ledger is compatible](04_phase_accepted.md#checking-the-ledger-is-compatible)) |
| `canBeCurrent` | whether a ledger is a sane successor of the validated one |
| `getValidatedLedger`, `getValidatedLedgerAge`, `haveValidated` | validated-ledger accessors |
| `storeLedger`, `getLedgerByHash` | cache a built ledger by hash, look one up |

It also holds the `LedgerHistory` (`LedgerHistory.h`), its store of recent ledgers indexed by sequence and hash.

`buildLedger` (`BuildLedger.h`) applies the agreed set to the parent and seals the immutable result ([Building the ledger](04_phase_accepted.md#building-the-ledger)). `OpenLedger` (`OpenLedger.h`) is the live open ledger. `accept` seeds the next round's ledger on the built one ([Seeding the next open ledger](04_phase_accepted.md#seeding-the-next-open-ledger)). Close-time arithmetic (`roundCloseTime`, `getNextLedgerTimeResolution`) lives in `LedgerTiming.h`.

## Acquisition

Consensus refers to ledgers and transaction sets by hash and often does not hold the contents. Fetching them from the network is a black box to the consensus algorithm: the adaptor's `acquireLedger` and `acquireTxSet` return what the node already holds, or return nothing and start a background fetch. The round never blocks. The data arrives later, over several network round-trips, and consensus picks it up on a subsequent tick. This fetching lives in the ledger and overlay layers, not the generic engine.

Both kinds of fetch share the same mechanics. The thing wanted is a `SHAMap`, pulled node by node from peers with `TMGetLedger` / `TMLedgerData` (objects also via `TMGetObjectByHash`), so a large tree takes many requests. The local node store is checked first. Only missing nodes go to the network. A fetch is driven by a `trigger` loop with three reasons, a peer was **added**, a peer **reply** arrived, or a **timeout** fired, and runs against a peer set: a timeout re-requests from other peers, so a slow or absent peer does not stall the fetch. Each request is capped to a bounded number of nodes.

### Ledgers (`InboundLedgers`)

A ledger does not arrive at once. `InboundLedger` fills it in three stages, each tracked by a flag:

- the **header** (`haveHeader_`): the ledger's identifying fields and the roots of its two trees.
- the **transaction tree** (`haveTransactions_`): the `SHAMap` of the ledger's transactions.
- the **account-state tree** (`haveState_`): the `SHAMap` of the ledger's state.

The ledger is complete once both trees are in. `InboundLedger::Reason` records why it is wanted: `CONSENSUS` (the current round needs it), `HISTORY` (backfilling past ledgers), or `GENERIC`.

From consensus, `acquireLedger` calls `InboundLedgers::acquireAsync(Reason::CONSENSUS)` and returns nothing. The round notices the ledger on a later tick: `checkLedger` switches to it once the node holds it ([Phase Dispatch](01_phase_dispatch.md)), and until then its validations sit in the `acquiring` map rather than the trie, where `getPreferred` can still fall back to them ([preferred_ledger.md](preferred_ledger.md)).

### Transaction sets (`InboundTransactions`)

A peer's position names its transaction set by hash. To compare positions and find disputes the node must hold the actual set, so a set it lacks is fetched the same way, by `TransactionAcquire`. A set is a single `SHAMap` (the transaction tree), so there are no stages: the one tree is pulled node by node until complete.

`getSet(setHash, acquire)` returns the set if held, or starts a fetch and returns nothing. When a fetch completes, `gotTxSet` feeds the set back into the round, recording the votes of every peer already proposing it ([Receiving a peer's position](03_phase_establish.md#receiving-a-peers-position)). `giveSet` stores a set the node already has (such as one it built), and `newRound` tells the subsystem a round has started so it can drop stale acquisitions.

| Concept | C++ |
|---|---|
| Ledger acquisition manager | `InboundLedgers` (`InboundLedgers.h`): `acquire` (blocking), `acquireAsync` (queued) |
| One in-flight ledger | `InboundLedger` (`InboundLedger.h`). Stage flags `haveHeader_`, `haveTransactions_`, `haveState_` |
| Why a ledger is wanted | `InboundLedger::Reason::{CONSENSUS, HISTORY, GENERIC}` |
| Transaction-set manager | `InboundTransactions` (`InboundTransactions.h`): `getSet`, `giveSet`, `newRound` |
| One in-flight set | `TransactionAcquire` (`detail/TransactionAcquire.h`) |
| Fetch driver | `trigger(peer, {Added, Reply, Timeout})`. Local-first via `tryDB` / `checkLocal` |
| Request / reply messages | `TMGetLedger` / `TMLedgerData`. Objects via `TMGetObjectByHash` |
| From consensus | `acquireLedger` -> `acquireAsync(Reason::CONSENSUS)`. `acquireTxSet` -> `getSet`, completion -> `gotTxSet` |

## Trust, quorum, and flag-ledger voting (`src/xrpld/app/misc/`)

`ValidatorList` (`ValidatorList.h`) owns the trusted validator set and the quorum: [`updateTrusted`](05_start_consensus_round_again.md#beginning-consensus) rebuilds both once per round, `setNegativeUNL` loads the prior ledger's negative UNL, and `negativeUNLFilter` drops disabled validators' validations. The full derivation is in [unl.md](unl.md).

Flag-ledger voting rides the same per-round cadence but lives in the adaptor and these helpers, not the generic code ([flag_ledger.md](flag_ledger.md)): `FeeVote` and `AmendmentTable` cast fee and amendment votes (`doValidation` attaches them to our validation, `doVoting` tallies them into the proposed set), and `NegativeUNLVote` scores validator reliability and emits the `ttUNL_MODIFY` pseudo-transactions. `AmendmentTable` also reports unsupported-amendment status, where `hasUnsupportedEnabled` drives the amendment block.

## Time, jobs, and the application container

`TimeKeeper` (`src/xrpld/core/TimeKeeper.h`) is the network clock: `closeTime()` is `now()` plus the damped `closeOffset_` drift correction (the network-adjusted time fed into consensus), and `adjustCloseTime` eases that offset toward the network each round ([Adjusting the close-time clock](04_phase_accepted.md#adjusting-the-close-time-clock)). `JobQueue` runs consensus work off the network threads: `JtNetopTimer` (heartbeat), `JtAccept` (accept-and-restart), `JtAdvance` (acquisition).

`Application` (`ApplicationImp`) is the top-level container: it owns `openLedger_` and `maxDisallowedLedger_` (the reboot anti-equivocation floor). The transaction queue `TxQ` holds fee-ordered transactions for future ledgers, applied when seeding the open ledger and pruned when a new ledger makes them stale.

