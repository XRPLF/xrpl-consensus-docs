# Terminology

**Ledger**: A ledger is a candidate or validated snapshot of XRPL state. It is the result of applying a set of transactions to the previous network state. Contains the transactions that were successfully applied in that step, and references the resulting state through a Merkle tree (SHAMap) backed by the node store. Identified by a sequence number, a hash, the parent hash, and a close time. 

**Open Ledger**: A mutable view layered on top of the most recent closed ledger. Accumulates incoming transactions between consensus rounds.

**Closed Ledger**: A ledger whose transaction set has been determined. No more transactions can be added. A closed ledger may or may not be validated - it becomes validated when enough trusted validations arrive to meet quorum.

**Last Closed Ledger (LCL)**: The most recent closed ledger. It serves as the starting point for building the next one.

**Validator**: A node configured with signing keys that actively participates in consensus by proposing positions and issuing validations. Only validators whose keys are in other nodes' UNLs influence the consensus outcome for those nodes.

**Unique Node List (UNL)**: The set of validators a node trusts. Assembled from keys listed directly in the node's configuration and from validator lists fetched from publisher sites and verified against trusted publisher. Recalculated at the start of each consensus round.

**Trusted Validator**: A validator whose public key appears in this node's UNL.

**Validated Ledger**: A closed ledger that has received enough trusted full validations to meet the quorum. Under normal network conditions, a validated ledger is not rolled back and the validated sequence only advances forward.

**Preferred Ledger**: The ledger this node should use as the prior ledger for consensus, the result of the Fork Choice. 

**Fork Choice**: The rule that decides which ledger to build the next one on when trusted validations point at competing branches. It walks the trie of validated-ledger ancestry and descends into a branch only while that branch's support stays ahead even if the still-undecided validators all defected, settling on the deepest such ledger. Its result is the Preferred Ledger. The walk is specified in [preferred_ledger.md](preferred_ledger.md).

**Transaction Set (TxSet)**: A set of transactions stored in a Merkle tree, identified by the tree's root hash. During consensus, each node proposes a transaction set, and nodes that propose different sets resolve the differences through dispute resolution.

**Proposal**:  A signed message broadcast by a validator during consensus. It carries the validator's current position: the transaction set hash it advocates, the proposed close time, and a sequence number that increases with each update. Validators may revise their position as the round progresses, or bow out if they detect they are on the wrong ledger. Sent during the consensus round as nodes work to agree on which transactions to include.

**Close Time**: The timestamp on a closed ledger. Since nodes have imperfect clocks, close times are rounded to a dynamic resolution (ranging from 10s to 120s) to make agreement feasible The raw close time is what validators propose and agree on during the consensus round. The effective close time is the value actually written to the ledger, which is the raw time rounded to the current resolution and guaranteed to be at least one second after the parent ledger's close time.

**Disputed Transaction**: A transaction that appears in one node's proposed transaction set but not another's. Discovered by acquiring and comparing the actual transaction sets behind different proposal hashes. Each trusted validator's position on the transaction (include or exclude) is inferred from whether the transaction exists in that validator's proposed set.

**Validation**: A signed message broadcast by a validator after it builds a new closed ledger, identifying the ledger by hash and sequence. A **full validation** indicates the validator was actively proposing during the round. A partial validation indicates it was observing or recovering. Only trusted full validations count toward the votes needed to mark a ledger as validated. Sent after the round completes and the resulting ledger is built.

**Quorum**: The minimum number of trusted full validations required for this node to consider a ledger fully validated. Under normal configuration, it is `ceil(max(0.8 * effectiveUNL, 0.6 * originalUNL))`, where `effectiveUNL = trusted validators - validators on the NegativeUNL`. Validations from NegativeUNL validators are filtered out before counting.

**Negative UNL**: A ledger entry listing validators that have been unreliable. Modified via pseudo-transactions at flag ledgers. These validators' validations are filtered out and they are excluded from quorum calculations. At most 25% of the UNL can be listed. Each proposing validator adds a pseudo-transaction to its transaction set to add a validator to the Negative UNL if it sent matching validations for fewer than 50% of the last 256 validated ledgers, or to remove it if that rate later exceeds 80%.

**Voting Ledger**: The ledger just before a flag ledger, where `(sequence + 1) % 256 = 0`. A validator's validation of a voting ledger carries its fee and amendment votes. Those votes are tallied into fee and amendment pseudo-transactions when the ledger after the flag ledger is built (see [flag_ledger.md](flag_ledger.md)).

**Flag Ledger**: A ledger whose sequence number is a multiple of 256. Negative-UNL pseudo-transactions are proposed in the round that builds it and can apply only there. Fee and amendment pseudo-transactions land in the ledger after it (see [flag_ledger.md](flag_ledger.md)).

**Round**: Three phases of consensus

**Consensus Phase**: Which of a round's three phases the node is in. `OPEN`: still collecting transactions, the ledger not yet closed. `ESTABLISH`: exchanging proposals with peers to converge on one transaction set and close time. `ACCEPTED`: the outcome is agreed and the ledger is being built. The phase does not change again until the next round starts.

**Consensus Mode**: How the node is taking part in the current round. `PROPOSING`: a validator broadcasting its own position. `OBSERVING`: following peer positions without proposing one. `WRONG_LEDGER`: building on the wrong prior ledger and trying to acquire the right one. `SWITCHED_LEDGER`: recovered to the correct ledger mid-round, now participating as if it had started observing.

**Consensus State**: The `ESTABLISH` phase's verdict on whether the round can finish. `No`: not enough agreement yet. `Yes`: we have consensus along with the network and can accept. `MovedOn`: the network reached consensus without us. `Expired`: the round hit its hard time limit.

**Sequence number**: Ledger sequence number (height)

**Proposal version**: Validator's proposal (also called sequence)

**Iteration**: Iteration of a phase within a round

**Avalanche state**: Current state of avalanche within an iteration
