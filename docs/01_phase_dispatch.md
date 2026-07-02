# Phase Dispatch

A consensus round advances through the `OPEN`, `ESTABLISH`, and `ACCEPTED` phases (see [terminology.md](terminology.md)). This doc covers what moves a round from one phase to the next.

Two things drive it:

* **The timer.** A periodic heartbeat ticks the round, deciding when to close the open ledger (`OPEN` -> `ESTABLISH`) and when peers have converged enough to accept (`ESTABLISH` -> `ACCEPTED`).
* **Completion.** Building the ledger at the end of the `ACCEPTED` phase starts the next round (see [Starting the Next Round](05_start_consensus_round_again.md)).

## Timer

Consensus is time-sensitive: the close and accept decisions above depend on elapsed time, and the node must also keep confirming it is still building on the ledger the rest of the network prefers. The **heartbeat**, a periodic timer, gives it a regular opportunity to make these decisions and to dispatch the handler for the current phase.

### Process Heartbeat Timer

`process_heartbeat_timer` combines the following responsibilities:
- It signals the **stall detector**, a separate thread that aborts the process if ten minutes pass without a signal. Because this routine is the only place that signal is sent, the consensus heartbeat doubles as the server's liveness check. 
- It sets the node's **operating mode**, its view of how connected and how in sync it is.
- If there are enough peers, it runs the actual timer entry.
- It arms itself to run again one second after each pass finishes, so its period is one second plus the time a pass takes. 

If it's connected to too few peers (`num_peers` < `min_peer_count`), it sets the mode to `DISCONNECTED` and arms itself to run again in one second.
* **min_peer_count**: defined by `network_quorum` configuration setting, defaults to 1. Not to be confused with the [consensus quorum](unl.md#quorum).
* **num_peers**: the size of all other connected nodes that completed the handshake, regardless of their UNL status and even the network id (for example, multiple isolated devnets can be identified by different network ids, but this value will contain any connected nodes).

If it runs while the node is in `DISCONNECTED` mode but now has enough peers, it sets the mode to `CONNECTED`.

In the C++ the `CONNECTED` and `SYNCING` cases look like no-ops, `setMode(mode)` calls with the mode already held. The decision sits inside `setMode`: it picks `SYNCING` or `CONNECTED` from the age of the last fully validated ledger, and drops back to `CONNECTED` while the node is amendment- or UNL-blocked.

```python
min_peer_count = 1
num_peers = Number of connected peers that completed the handshake 
mode = OperatingMode

def process_heartbeat_timer():
    notify_stall_manager()
    
    if num_peers < min_peer_count:
        if mode != DISCONNECTED:
            mode = DISCONNECTED
        arm(process_heartbeat_timer, 1s)
        return
        
    if mode == DISCONNECTED:
        mode = CONNECTED
        
    can_sync = (fully_validated_ledger_age() < 1min) and not (amendment_blocked or unl_blocked)
            
    if mode in [CONNECTED, SYNCING]:
        mode = SYNCING if can_sync else CONNECTED
    
    timer_entry()
    arm(process_heartbeat_timer, 1s)
```

### Timer Entry

If there are enough peers, the heartbeat calls timer entry. `startRoundInternal` also calls it at round start when more than half the previous round's proposers have already sent positions (see [Starting the Next Round](05_start_consensus_round_again.md)). Timer entry is the generic engine's tick (`Consensus::timerEntry`), and everything it dispatches (`check_ledger`, the `OPEN` and `ESTABLISH` handlers) is generic too. `RCLConsensus::timerEntry` is only the RCL lock wrapper around it. It rethrows an exception that can only arise from data corruption.

The tick lets the engine make progress that depends on the passage of time rather than on a new message. Without it the engine would only ever react to incoming proposals, and a round could stall. Each tick first refreshes the engine's notion of the current network time, then confirms the node is still on the right prior ledger, then advances whichever phase is active.

* `current_server_time` is the server's wall clock, expressed as time elapsed since the epoch start (2000-01-01).
* `close_offset` is updated at the end of each round: the vote-weighted average of the round's close-time votes (ours included) minus our own close time, applied as a damped step ([Adjusting the close-time clock](04_phase_accepted.md#adjusting-the-close-time-clock)). It nudges our clock toward the network's consensus close time.

```python
phase = Current consensus phase
network_adjusted_time = Network adjusted time # Consensus.h now_
current_server_time = Wall clock
close_offset = Drift of our own clock to network's average

def timer_entry():
    if phase == ACCEPTED:
        return
    
    network_adjusted_time = current_server_time + close_offset
    
    check_ledger()

    if phase == OPEN:
        phase_open()
    elif phase == ESTABLISH:
        phase_establish()
```

Timer entry does nothing in the `ACCEPTED` phase: the outcome is already agreed, and building the ledger runs on its own and finishes by starting the next round.

`check_ledger` makes sure we are still building on the ledger the network prefers, and that we actually hold it. It asks the fork choice for the preferred prior ledger. If that is the one we are already building on and we hold it, there is nothing to do.

Otherwise we are on the wrong ledger. We bow out of the round (stop proposing). If the preferred ledger differs from the one we were targeting, we retarget to it, discard the round's working state, and replay any proposals we had buffered for the new ledger. If we now hold the target ledger we are done. If not, and it is available locally or from a peer, we adopt it and restart the round on it (`SWITCHED_LEDGER`). If it is not yet available, we fetch it asynchronously and switch on a later tick (`WRONG_LEDGER`).

* **leave_consensus**: stop proposing our position and bow out of the current round.
* **discard_round_state**: clear this round's working state. These are the disputed transactions, the peers' positions, their close-time votes, and the nodes marked unresponsive.
* **replay_buffered_proposals**: re-process positions peers already sent for the preferred ledger. Peers ahead of us may have broadcast positions for it before we switched. Those were buffered without counting toward the old round, so replaying them seeds the new round at once instead of waiting for the next broadcast.

```python
prior_ledger = the ledger this round is building the next one on top of
mode = ConsensusMode

def check_ledger():
    preferred = preferred_prior_ledger()           # fork choice over trusted validations
    if preferred == prior_ledger and we_hold(prior_ledger):
        return

    leave_consensus()
    if preferred != prior_ledger:
        prior_ledger = preferred
        discard_round_state()
        replay_buffered_proposals()
    if we_hold(prior_ledger):
        return
    if available(prior_ledger):
        switch_to(prior_ledger)
        mode = SWITCHED_LEDGER
    else:
        mode = WRONG_LEDGER
```

`fully_validated_ledger_age` gives the age of our last fully validated ledger, used by the heartbeat to decide whether we are synced. It subtracts that ledger's sign time (the median of its validators' sign times, recorded when we fully validated it) from the network-adjusted time. With no fully validated ledger it returns a large value, so the node does not count itself synced.

```python
def fully_validated_ledger_age():
    if valid_ledger_sign_time is unset:         # no fully validated ledger yet
        return 2 weeks                          # large, so the node is not considered synced
    return max(network_adjusted_time - valid_ledger_sign_time, 0)
```

## C++ reference

| Pseudocode | C++ |
|---|---|
| **`process_heartbeat_timer`** | `NetworkOPsImp::processHeartbeatTimer`. Signals the stall detector, sets the operating mode (`setMode`), and with enough peers runs the timer entry |
| `mode` (operating) | `NetworkOPsImp::mode_` |
| `min_peer_count` | `minPeerCount_` (from the `network_quorum` config) |
| `num_peers` | `Overlay::size()` |
| **`timer_entry`** | `RCLConsensus::timerEntry` (lock-guard) over `Consensus::timerEntry`. Refreshes `now_`, calls `checkLedger`, dispatches to `phaseOpen`/`phaseEstablish` |
| `phase` | `Consensus::phase_` |
| `network_adjusted_time` | `Consensus::now_` |
| `current_server_time` | `TimeKeeper::now()` (wall clock at the XRPL epoch) |
| `close_offset` | `TimeKeeper::closeOffset_` |
| **`check_ledger`** | `Consensus::checkLedger` -> `handleWrongLedger` |
| `prior_ledger` | `Consensus::previousLedger_` (id `prevLedgerID_`) |
| `mode` (consensus) | `Consensus::mode_` |
| `preferred_prior_ledger` | fork choice (`Validations::getPreferred`), see [preferred_ledger.md](preferred_ledger.md) |
| `leave_consensus` | `Consensus::leaveConsensus` |
| **`fully_validated_ledger_age`** | `LedgerMaster::getValidatedLedgerAge` (uses `validLedgerSign_`, the validated ledger's median sign time) |
