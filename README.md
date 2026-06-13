# Gossip Suspicion

A **suspicion timeout and state-transition engine** for SWIM-style failure detection — managing the critical window between "unresponsive" and "declared dead" with tunable timeouts, suspicion sub-protocol messaging, and incarnation-based refutation.

## Why It Matters

Declaring a node dead is irreversible in most systems — it triggers rebalancing, data redistribution, and alerting. False positives (declaring a healthy node dead) are extremely costly: data may be duplicated, connections dropped, and cascading failures triggered. This library manages the suspicion window: when a node stops responding to pings, it enters SUSPECT state for a configurable timeout period. During this window, the suspected node can refute the suspicion by broadcasting its liveness with a higher incarnation number. Only if no refutation arrives before the timeout does the node transition to DEAD. This timeout-based "guilty until proven innocent" approach dramatically reduces false-positive rates compared to immediate death declarations.

## How It Works

**Suspicion lifecycle**:
```
ALIVE ──(ping timeout)──→ SUSPECT ──(T_suspect)──→ DEAD
                               ↑                       │
                               │   (refute: inc+1)     │
                               └─── ALIVE (inc bumped)  │
                                                       │
                                          (T_reap) ────┘
                                          Remove from member list
```

**Timeout calculation**: The suspicion timeout T_suspect must account for:
- Network round-trip time (RTT) for the refutation message to propagate
- Gossip fanout depth (O(log N) rounds for full dissemination)
- Clock skew between nodes (bounded if using NTP)

A common heuristic:
```
T_suspect = T_probe × suspicion_mult × log2(N) + safety_margin
```
where `suspicion_mult` is typically 4–6 and `safety_margin` is 500ms–2s.

For N=100, T_probe=1s: T_suspect = 1 × 5 × 7 + 1 = 6s.

**Refutation protocol**: When a node discovers it's suspected (via gossip), it immediately:
1. Increments its own incarnation number: `inc ← inc + 1`
2. Broadcasts an ALIVE message with the new incarnation: `{node: self, state: ALIVE, inc: new_inc}`
3. Peers receiving this message apply incarnation arbitration: since `new_inc > suspect_inc`, the ALIVE state overrides SUSPECT

**Incarnation arbitration** is the correctness invariant:
- For equal incarnations: state precedence is DEAD > SUSPECT > ALIVE (pessimistic)
- For unequal incarnations: higher incarnation always wins, regardless of state

This prevents a stale SUSPECT message from overriding a fresh ALIVE refutation.

**Suspicion sub-protocol (from Lifeguard)**: Instead of a fixed timeout, each node adjusts its local suspicion timeout based on how many other nodes have confirmed the suspicion. More confirmations → shorter timeout (the node is probably really dead). No confirmations → longer timeout (could be a false positive). This adaptive mechanism further reduces false positives.

**Complexity**:
- Start suspicion: O(1) — HashMap insert
- Check timeout: O(K) per round — scan suspected nodes
- Process refutation: O(1) — HashMap lookup + incarnation compare
- Memory: O(N) — one timer per suspected node

## Quick Start

```rust
use gossip_suspicion::{SuspicionEngine, SuspicionConfig};

let config = SuspicionConfig::new()
    .suspicion_mult(5.0)
    .safety_margin_ms(1000);

let mut engine = SuspicionEngine::new(config);

// Mark node as suspect
engine.start_suspicion("node-B", incarnation: 3);

// Check for timeouts
let expired = engine.check_timeouts();
for node in expired {
    println!("{} confirmed dead", node);
}

// Process refutation from node-B
engine.refute("node-B", 4); // higher incarnation → back to ALIVE
```

## API

| Type | Description |
|------|-------------|
| `SuspicionEngine::new(config)` | Create the suspicion state machine |
| `SuspicionConfig` | Tuning (multiplier, safety margin) |
| `.start_suspicion(node, inc)` | Begin suspicion timeout for a node |
| `.check_timeouts()` | Return nodes whose timeout has expired → DEAD |
| `.refute(node, new_inc)` | Process refutation: cancel suspicion if inc is higher |
| `.is_suspected(node)` | Check if a node is currently suspected |

## Architecture Notes

Gossip Suspicion handles the most safety-critical phase of the SuperInstance gossip stack — the transition from "maybe dead" to "definitely dead." The adaptive timeout mechanism directly controls the γ/η trade-off in **γ + η = C**: shorter timeouts reduce detection latency (lower γ) but increase false positives (higher η overhead from unnecessary rebalancing). See [Architecture](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## References

- Das, A. et al. "SWIM: Scalable Weakly-consistent Infection-style Membership," DSN (2002).
- Oustehout, J. et al. "Lifeguard: SWIM-ing with Situational Awareness," DSN (2019).
- Clement, A. et al. "Making Gossip Robust to Byantine Faults," ICDCS (2013).

## License

MIT
