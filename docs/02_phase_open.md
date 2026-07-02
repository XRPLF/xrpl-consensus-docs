# OPEN Phase

In the `OPEN` phase, a validator accumulates incoming transactions into its open ledger and decides when to close it. Closing too early leaves no time for transactions to arrive or for the close time to settle. Waiting too long lets the node fall behind the network. When the close condition is met, it closes the ledger and the round advances to the `ESTABLISH` phase.

## Terminology

* **Open transactions**: Incoming transactions, accumulated in the open ledger. Throughout the consensus round, a validator collects all transactions it saw. It keeps them in open ledger, and in the `ACCEPTED` phase it applies them to the newly built ledger.
* **Previous ledger**: The last closed ledger, the parent of the ledger this round is building on. Either the last ledger the consensus round built, or a [preferred ledger](preferred_ledger.md).
* **Current peer positions**: Positions from trusted UNL validators for the current consensus round - i.e. proposals building on the same **prev_ledger**'s ID (hash). Cleared at the start of a consensus round. Contains proposals of peers who have not bowed out.
* **Close time resolution**: The bin size that ledger close times are rounded to.
* **Close time agreement**: Whether a consensus was reached on the close time. 
* **Close time**: Binned ledger close time that the network agreed on. (stored in `previousLedger_`'s header, not to be confused with `prevCloseTime_`).
* **Local close time**: Node's own observed close time of previous ledger: wall clock + damped `closeOffset` drift correction (`prevCloseTime_`), at one second precision.
* **Network adjusted time**: Wall clock + damped `closeOffset` drift correction ([Adjusting the close-time clock](04_phase_accepted.md#adjusting-the-close-time-clock)). It does not tick automatically, it is advanced forward on various events.  
* **Monotonic clock**: A clock ticking away without regard for timezones or leap seconds or years. Like a stop watch. 

## Overview

On each timer tick in the `OPEN` phase, the node decides whether to close the open ledger. It closes when any of these holds:

- **Timing**: the previous round's duration or the time since the last close is implausible (negative, or over ten minutes), so we close to resynchronize.
- **A majority has moved on**: more than half of the previous round's proposers have already closed this round or full-validated the previous ledger.
- **Idle long enough**: with no open transactions, the time since the last close has reached the idle interval (the larger of `ledger_idle_interval` and twice the previous close-time resolution).
- **Open long enough with work**: there are open transactions, the minimum open time has passed, and we are not closing faster than half the previous round took.

The close decision runs in the generic engine.

```python
has_open_transactions = There are some open transactions
proposers_closed = The number of proposers who appear in current_peer_positions 
open_phase_start = Monotonic clock value when we last entered the OPEN phase
clock_now = Monotonic clock current value
last_ledger_full_validations = Number of proposers who submitted a **full** validation for the prev_ledger_id
prev_proposers = The number of proposers in the previous consensus round
prev_round_time = How long the previous consensus round took to converge
ledger_idle_interval = 15 seconds
ledger_min_close = 2 seconds
previous_ledger = The previous ledger
previous_ledger_parent = Parent of the previous ledger
network_adjusted_time = Network adjusted time
mode = Consensus mode
local_previous_close_time = Local close time 


def phase_open():
    open_time = clock_now - open_phase_start
    
    previous_close_correct = 
        mode != WRONG_LEDGER 
        and previous_ledger[close time agreement] 
        and previous_ledger[close time] != previous_ledger_parent[close time] + 1 second

    last_close_time = previous_ledger[close time] if previous_close_correct else local_previous_close_time
    time_since_previous_close = network_adjusted_time - last_close_time if network_adjusted_time >= last_close_time else -(last_close_time - network_adjusted_time)
    idle_interval = max(ledger_idle_interval, 2 * previous_ledger[close time resolution])
    
    unexpected_timing = prev_round_time < -1 second or prev_round_time > 10 minutes or time_since_previous_close > 10 minutes
    majority_closed = proposers_closed + last_ledger_full_validations > prev_proposers / 2
    idle_timeout_met = time_since_previous_close >= idle_interval
    min_open_met = open_time >= ledger_min_close
    not_too_fast = open_time >= prev_round_time / 2

    if unexpected_timing 
        or majority_closed
        or (not has_open_transactions and idle_timeout_met)
        or (has_open_transactions and min_open_met and not_too_fast):
        close_ledger()   # defined in 03_phase_establish.md
```

## C++ reference

| Pseudocode | C++ |
|---|---|
| **`phase_open`** | `Consensus::phaseOpen`. The close decision is the free function `shouldCloseLedger` (`Consensus.cpp`), and on close it calls `closeLedger`, which invokes the adaptor's `onClose` |
| `has_open_transactions` | `RCLConsensus::Adaptor::hasOpenTransactions()` |
| `proposers_closed` | `currPeerPositions_.size()` (`Consensus::currPeerPositions_`) |
| `open_phase_start` | `Consensus::openTime_` |
| `clock_now` | `Consensus::clock_.now()` |
| `last_ledger_full_validations` | `RCLConsensus::Adaptor::proposersValidated(prevLedgerID_)` |
| `prev_proposers` | `Consensus::prevProposers_` |
| `prev_round_time` | `Consensus::prevRoundTime_` |
| `ledger_idle_interval` | `ConsensusParms::ledgerIdleInterval` |
| `ledger_min_close` | `ConsensusParms::ledgerMinClose` |
| `previous_ledger_parent` | `previousLedger_.parentCloseTime()` |
| `local_previous_close_time` | `Consensus::prevCloseTime_` |
