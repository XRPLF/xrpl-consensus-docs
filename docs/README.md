# XRP Ledger Consensus

Consensus is how the XRP Ledger stays a single shared ledger across a decentralized network with no central coordinator. 
The validators that maintain the ledger must agree on each new ledger: which transactions it contains, in what order, and when it closed. Consensus is the process that reaches that agreement. Round after round, validators propose a next ledger and exchange and revise their positions until a supermajority of trusted validators converge on one result, which each then ratifies with a signed validation. This keeps honest nodes advancing the same ledger history even when some participants are slow, unreachable, or dishonest.

## The Protocol

Each node runs a continuous sequence of **consensus rounds**, and each round appends one ledger to the chain.

A round has to settle what the next ledger will contain and the time it closed. The **transaction set** is the exact group of transactions the new ledger applies, in a fixed order. A node gathers the transactions it has heard from clients and peers, but transactions reach different nodes at slightly different times, so nodes can begin a round holding different pending sets. The **close time** is the single timestamp recorded on the ledger. Each node reads its own imperfect clock, so it cannot simply use that reading. Reconciling these into one transaction set and one close time that every honest node adopts is what the phases below do.

A round moves through these phases:

* **`OPEN`**: the node collects transactions into a candidate ledger.
* **`ESTABLISH`**: it exchanges positions with trusted peers and converges on one transaction set and one close time, settling each disagreed transaction through dispute voting.
* **`ACCEPTED`**: the agreed set is applied to build the new ledger, which the node then signs a validation for.

A roughly one-second **heartbeat** drives the time-based decisions: when to close the open ledger, and when peers have agreed closely enough to accept. Finishing a round starts the next one on top of the ledger just built.

Agreement counts only among **trusted validators**, the validators on a node's UNL. A ledger becomes **fully validated** once a **quorum** of trusted validators have signed a validation for it. Validation runs continuously alongside the rounds, so a ledger can reach full validation between a node's own rounds as peers' validations arrive.

When trusted validations point at competing branches, the **preferred-ledger** rule decides which ledger a new round builds on, so nodes that briefly diverge reconverge on one chain.

The algorithm itself is written generically, over abstract ledgers, transaction sets, and positions, and contains no XRP Ledger specifics. The surrounding code supplies the concrete types and drives it. 

## Document Organization

The numbered docs follow a single consensus round in the order it happens: the timer tick that drives the phase changes, then the `OPEN`, `ESTABLISH`, and `ACCEPTED` phases, then the handoff that starts the next round. Read in order, 01 through 05, they make up the full algorithm:

| Doc | Covers |
|---|---|
| [01_phase_dispatch.md](01_phase_dispatch.md) | the heartbeat and timer tick that drive the phases, and the check that we are still on the network's preferred prior ledger |
| [02_phase_open.md](02_phase_open.md) | the `OPEN` phase: collecting transactions and deciding when to close |
| [03_phase_establish.md](03_phase_establish.md) | the `ESTABLISH` phase: exchanging positions, dispute voting, the close-time vote, and the consensus check |
| [04_phase_accepted.md](04_phase_accepted.md) | the `ACCEPTED` phase: building the ledger, signing the validation, and advancing the fully validated ledger |
| [05_start_consensus_round_again.md](05_start_consensus_round_again.md) | ending the round and starting the next: rebuilding trust and quorum, and deciding whether to propose |

Starting a round could sit at either end of this sequence. We put it last because much of what it does, resetting per-round state, only makes sense once you have read the earlier phases and seen what state a round accumulates.

Some topics are large enough to warrant their own pages:

* [preferred_ledger.md](preferred_ledger.md): the fork choice, how trusted validations pick which ledger to build on.
* [unl.md](unl.md): the trusted validator set, the quorum, and the negative UNL.

[flag_ledger.md](flag_ledger.md) covers the periodic ledgers that carry fee, amendment, and negative-UNL voting. Flag ledgers are not crucial to the functioning of consensus, but they are important to understanding how Negative UNL works and are coupled to parts of the consensus code.

Censorship and byzantine detections do not impact how consensus reaches agreement, but learning about them could prove useful to validator operators who want to understand potential issues surfaced in the logs:

* [censorship_detector.md](censorship_detector.md): flags transactions repeatedly left out of accepted ledgers.
* [byzantine_detector.md](byzantine_detector.md): flags a validator that signs more than one validation for the same sequence.

**Reference:**

* [code_organization.md](code_organization.md): the code map, the generic algorithm and the RCL binding, the classes and files, and the messages, ledger, trust, and acquisition subsystems around them.
* [terminology.md](terminology.md): the glossary of terms used throughout.

## Where to start

New readers should skim [terminology.md](terminology.md), then read the numbered docs 01 through 05 in order, following the concept links as they come up. Readers mapping the specification to the source code should read [code_organization.md](code_organization.md).
