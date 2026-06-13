# Gossip Suspicion

**A Rust library implementing the suspicion sub-protocol of SWIM gossip** — a timeout-based mechanism that transitions nodes from suspect to dead, with incarnation-number conflict resolution to prevent false positives.

## Why It Matters

In gossip protocols, a missed ping doesn't necessarily mean a node has crashed — network partitions, GC pauses, and packet loss can all cause temporary unreachability. The suspicion sub-protocol adds a grace period: instead of immediately declaring a node dead, it's marked "suspect" for a configurable timeout. If the node refutes the suspicion (by sending a higher incarnation number), it returns to healthy. This dramatically reduces false-positive failure detections in flaky networks. Production systems like HashiCorp memberlist use an exponential timeout so that large clusters don't generate a thundering herd of suspicions simultaneously.

## How It Works

When a node's probes fail, the local node marks it `Suspect` with the current incarnation number and starts a suspicion timer. The suspicion is disseminated via gossip. If the suspected node receives the suspicion message, it increments its incarnation and broadcasts a "refutation" (alive with higher incarnation). If no refutation arrives before the timeout expires, the node is declared `Dead`. The timeout is typically randomized and may use a protocol like Lifeguard's adaptive timeout (longer for nodes with more recent activity). Incarnation numbers prevent stale gossip messages from overriding newer state: a lower-incarnation message is always discarded.

## Quick Start

```rust
// API surface under development — the crate currently provides
// foundational types for suspicion timeout tracking.
use gossip_suspicion::add;

fn main() {
    assert_eq!(add(2, 2), 4);
}
```

## API

| Function | Description |
|---|---|
| `add(left, right)` | Placeholder — full suspicion API under development |

## Architecture Notes

Part of the SuperInstance gossip stack: `gossip-protocol`, `gossip-member`, `gossip-ping`, `gossip-seed`, `gossip-suspicion`. See the [Architecture Guide](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## License

MIT
