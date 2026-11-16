# Raft Consensus Lab

An interactive, in-browser simulator of the [Raft consensus algorithm](https://raft.github.io/), the protocol that keeps etcd (and therefore every Kubernetes control plane) consistent when servers crash or the network splits.

Five simulated servers elect a leader and replicate a shared log. You can crash servers, cut them off the network and send client writes, and watch the cluster recover in real time.

**Live demo:** `https://<your-username>.github.io/raft-consensus-lab/` (after enabling GitHub Pages, see below)

## What you can do

| Action | What it shows |
| --- | --- |
| Send a client write | Log replication: the leader copies the entry to followers and commits it once a majority has it |
| Stop a server / stop the leader | Leader election: a follower times out, asks for votes and takes over in a higher term |
| Cut a server off the network | Network partitions: a cut-off leader keeps accepting writes it can never commit |
| Make a server call an election | Term changes and vote rules, on demand |
| Repair everything | Recovery: stale servers step down and catch up from the leader |

The page also shows each server's copy of the log (filled = committed, dashed = not yet safe, colour = term) and a plain-English feed of every election, step-down and commit.

## Three experiments to try

1. **Crash the leader.** A follower runs out of patience, wins an election, and no committed write is lost.
2. **Lose the majority.** Stop three of five servers. The remaining two can never win an election, so nothing new commits. Raft chooses to be unavailable rather than wrong.
3. **Split brain.** Cut the leader off and send it two writes. It accepts them but cannot commit them. The other four elect a new leader. Reconnect the old leader: it steps down, and on the next write it discards its uncommitted entries to match the new leader.

## How it works

Everything is in one file, `index.html`: plain JavaScript and a `<canvas>`, no framework, no build step, no dependencies.

The simulation runs on a virtual clock. Each server holds the state from the Raft paper (`term`, `votedFor`, `log`, `commit`, and for leaders `next` / `match` per follower). Messages are objects that take 420 to 580 ms of virtual time to travel, so you can see them in flight.

| Function | Raft concept |
| --- | --- |
| `startElection` | Follower times out, becomes candidate, increments term, requests votes |
| `deliver` (`voteReq`) | One vote per term, and only for a candidate whose log is at least as up to date |
| `becomeLeader` | Majority of votes wins; leader initialises `next` and `match` |
| `sendAppend` / `broadcast` | AppendEntries RPC, also used as the heartbeat |
| `deliver` (`append`) | Consistency check on `prevIndex` / `prevTerm`; conflicting entries are truncated |
| `deliver` (`appendRes`) | Leader advances the commit index when a majority holds an entry from its own term |
| `stepDown` | Any server that sees a higher term becomes a follower |

Timings: heartbeat every 1.1 s, election timeout randomised between 3 s and 6 s (slowed down from real systems so it is watchable).

## Simplifications

This is a teaching tool, not a production implementation. Compared with the paper:

- A restarted server keeps its commit index (in real Raft it is volatile and rebuilt from the leader).
- A new leader does not append a no-op entry, so a stale server's uncommitted entries are only removed on the next client write.
- No pre-vote: a reconnected server with a higher term can force a fresh election.
- No log compaction, snapshots or membership changes.
- When a follower rejects an append, the leader backs up one entry at a time.

## Run it locally

Open `index.html` in a browser. That is all.

## Deploy on GitHub Pages

1. Push this folder to a repository named `raft-consensus-lab`.
2. In the repository, go to **Settings → Pages**.
3. Under **Source**, choose **Deploy from a branch**, select `main` and `/ (root)`, then save.
4. After a minute the demo is live at `https://<your-username>.github.io/raft-consensus-lab/`.

## Reference

Diego Ongaro and John Ousterhout, *In Search of an Understandable Consensus Algorithm* (2014): https://raft.github.io/raft.pdf

## License

MIT, see [LICENSE](LICENSE).
