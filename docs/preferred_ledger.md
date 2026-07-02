# Choosing the Preferred Ledger

Network delay and partitions can leave trusted validators validating competing branches of the ledger history. To re-converge, a node must deterministically pick the **preferred ledger** to build its own ledger on and potentially abandon its current branch whenever that pick differs.

## Finding the Preferred Ledger

The node keeps receiving Validations from trusted validators (recorded as they arrive, see [Incoming validations](04_phase_accepted.md#incoming-validations)), and each prompts it to fetch the actual Ledger behind it from the network.
They do not need to be full Validations. The trusted set is rebuilt at the start of each round (see [`begin_consensus`](05_start_consensus_round_again.md#beginning-consensus)). `Validations::trustChanged` then adds or drops those validators' support in the trie.
Each acquired Ledger is inserted into the `LedgerTrie`: its skip-list ancestry (the most recent 256 parent hashes) places it on the correct branch, and the trusted validator backing it is counted as support.

Take for example that the node has validated up to and including sequence N + 2. Including itself, it has heard validations from 10 trusted validators. The preferred ledger information would look like this:

```
                                root
                                  |
seq N (uncommitted: 1):       Ledger A (votes: 10)
                                /              \ 
seq N + 1 (1):             AA (8)            AB (2)           
                             |                  |                  
seq N + 2 (1):             AAA (7)           ABC (2) 
                           /   \
seq N + 3 (5)        AAAA (1) AAAB (4)                                        
```

The trie answers the question: at every sequence, how many, of the known trusted validators, have voted for a particular ledger (i.e. support a particular branch).
Those validators, that have been seen, but who have no vote for a ledger at a sequence, are **uncommitted**.

We count a validator as **uncommitted** when it lags behind - behind the sequence we're examining, or behind how far this node has itself validated, whichever is further ahead.

In the example above, there is a total of 10 accumulated Validations from trusted peers. 
* At sequence N, branch support is unanimous. However, one validator is lagging behind us (we have validated up to N + 2, they have validated up to N + 1), so we add it to uncommitted count.  
* At sequence N + 1, votes favor ledger AA.
* At sequence N + 2, votes favor ledger AAA. One vote is now missing, but that vote cannot change the outcome. 
* At sequence N + 3, votes favor ledger AAAB. However, 5 votes are uncommitted - and they could all go to AAAA. The fork test is: `(leader - rival + tie) > uncommitted`.  

Because AAA is the highest ledger that cannot, even after receiving missing votes, switch to another branch, ledger AAA and sequence N + 2 is taken as the preferred ledger.

A branch is **safe** to step onto only if its lead survives even if every uncommitted validator defected to its strongest rival. The preferred ledger is the deepest ledger reachable by safe steps.

When two candidates have equal support, the one with the larger hash wins, so the **tie-break** is deterministic. When the leader owns the tiebreak, + 1 is added to the margin in the fork test: `(leader - rival + 1) > uncommitted`, so the rival would have to strictly exceed it to win.

The preferred ledger choice is handled by `pseudocode/ledger_trie.py`

### The walk, step by step

| Seq  | Leading ledger | Branch support | Uncommitted | Margin   | `margin > uncommitted`? | Action      |
|------|----------------|---------------:|------------:|---------:|:-----------------------:|-------------|
| N    | A              | 10             | 1           | 10       | 10 > 1 ✓                | descend     |
| N+1  | AA             | 8              | 1           | 6        | 6 > 1 ✓                 | descend     |
| N+2  | AAA            | 7              | 1           | 7        | 7 > 1 ✓                 | descend     |
| N+3  | AAAB           | 4              | 5           | 4 (3+1)  | 4 > 5 ✗                 | **stop**    |

The walk descends A -> AA -> AAA, then stops at the N+3 fork, leaving **AAA (seq N+2)** as the preferred ledger.
In N + 3, margin is 3 + 1, because the lead is 3, but AAAB wins tie break over AAAA, so +1 is added to it.

Caveats on the trie behind these counts:

- **Stale validations are flushed lazily.** Before each read to determine preferred ledger, node drops any validation no longer `isCurrent` (signTime older than ~3 min) from the trie. It runs on read, not on a timer, so a validator whose Validations we haven't seen in 3 minutes is taken out of consideration.
- **Sequence disjunction.** Ancestry comes from the skip list (the 256 most recent hashes), so two ledgers more than 256 sequences apart are not recognized as related even on the same chain. `B` descends from `A` only when `B.seq - A.seq <= 256`. This rarely has real effect because of stale Validation handling.
- **Empty trie (cold start).** The trie only holds ledgers we have a local copy of, while Validations for ledgers we haven't fetched yet are in an `acquiring` map. A starting or far-behind node can have an empty trie, so `getPreferred` falls back to the `acquiring` ledger with the most trusted validators (tie-break by larger hash).

## How a node handles the Preferred Ledger

Having found the preferred tip, the node compares it  to the ledger it's currently building on:

- If the preferred tip is simply the **next ledger on our own chain**, we're already about to build it, no switch needed.
- If it's **further ahead**, we've fallen behind, so we jump forward to it.
- If it's at our height or behind but on a **different chain**, we built on the wrong branch, so we switch to the preferred one.
- Otherwise we're already on the preferred chain and stay.

All of this is floored at the node's **fully-validated ledger**: `getPrevLedger` calls the bounded `getPreferred(curr, minValidSeq)` with `minValidSeq = getValidLedgerIndex()`, so it never switches to a preferred ledger below that tip.
A fully-validated ledger is one that reached quorum of trusted validators.

The check for preferred ledger is run every second, from `Consensus::timerEntry` which calls `Consensus::checkLedger` (the per-tick consult of the preferred prior ledger, see [Timer Entry](01_phase_dispatch.md#timer-entry)). `Adaptor::getPrevLedger` calls `Validations::getPreferred`, which in turn calls `LedgerTrie::getPreferred` and then runs the logic to see what to do with the preferred ledger.
If the node is on the wrong ledger, `Consensus::checkLedger` will handle it in `Consensus::handleWrongLedger`.

## C++ reference

| Concept | C++ |
|---|---|
| Preferred-ledger walk over the trie | `LedgerTrie::getPreferred` |
| Tip / branch support counts | `LedgerTrie::tipSupport`, `LedgerTrie::branchSupport` |
| Fork choice over trusted validations, plus the switch/stay decision | `Validations::getPreferred(curr)` |
| Same, floored at the validated sequence | `Validations::getPreferred(curr, minValidSeq)` |
| Stale-validation flush before each trie read | `Validations::withTrie` over `Validations::current` (via `isCurrent`) |
| Cold-start fallback to the acquiring ledger | `Validations::getPreferred(curr)` (acquiring branch) |
| Trust changes add or drop trie support | `Validations::trustChanged` |
| Skip-list ancestry of a validated ledger | `RCLValidatedLedger::operator[]`, `mismatch` |
| Per-tick consult and wrong-ledger handling | `Consensus::checkLedger`, `Consensus::handleWrongLedger` |
| Adaptor binding that calls the fork choice | `RCLConsensus::Adaptor::getPrevLedger` |
| Validated-ledger floor | `LedgerMaster::getValidLedgerIndex` |
