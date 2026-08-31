# Local patches carried by this fork

This is a fork of [`asternic/wuzapi`](https://github.com/asternic/wuzapi) (MIT).

**Default state of this fork is: no patches — byte-identical to upstream.**
Every entry below is a temporary divergence that exists *only* while an
upstream pull request is open. A patch without an upstream PR is not allowed
here; if you find one, it is a bug in our process, not a precedent.

When a patch is merged upstream, the next upstream sync brings it in as an
upstream commit, the local patch is dropped, and **its entry is removed from
this file**.

---

## Active patches

### 1. Serialize access to the shared AMQP channel (`rabbitmq.go`)

| | |
|---|---|
| **Commit** | `3003228f926d4576f5306c87227d2b550b73f370` |
| **Branch** | `fix/rabbitmq-concurrent-publish` |
| **Base** | `e2bdc1b5fbb2a39da6ffbd30b0accfbb2430d5c6` (upstream merge point, v1.0.8) |
| **Files** | `rabbitmq.go`, `rabbitmq_test.go` (new) |
| **Upstream issue** | https://github.com/asternic/wuzapi/issues/362 |
| **Upstream PR** | https://github.com/asternic/wuzapi/pull/363 |
| **Applied** | 2026-08-31 |
| **Re-evaluation deadline** | **2026-11-29** (90 days) |

**Why.** `PublishToRabbit` used one package-level `amqp091.Channel` from
several goroutines with no synchronization. The two calls it makes on that
channel are guarded differently by the library: `Channel.Publish` holds the
channel's internal mutex (`ch.m`) while it writes the method, header and body
frames, but `Channel.QueueDeclare` (via `ch.call()`) does not take that mutex
at all. Because every publish declares the queue first, a `Queue.Declare` frame
can be written in the middle of another goroutine's content frames on the same
channel id. The broker then closes the connection with
`unexpected_frame: "expected content body, got non content body frame instead"`
and every event in flight is lost.

Two defects of the same family are fixed with it: concurrent `QueueDeclare`
calls consuming each other's `Queue.DeclareOk`, and the unsynchronized swap of
the connection state during reconnection.

Measured impact before the fix: 2184 events lost across 6 windows in one day,
with the largest burst matching the day's peak event volume.

**Scope.** Transport only. No payload, wire format or HTTP response shape is
touched, and the patch carries nothing specific to any particular deployment —
it is a generic concurrency fix, written to be accepted upstream.

**On an upstream sync while this is open:** rebase the patch onto the new base.
If the rebase conflicts in a non-trivial way, stop and re-evaluate rather than
resolving it by hand — a costly rebase is the signal that carrying the patch
has stopped being cheap.

**Outcomes.** If upstream merges it, drop the patch and remove this entry. If
upstream fixes the defect differently, adopt their fix and drop ours. If there
is no outcome by the deadline above, the patch does not become permanent by
default — it gets re-decided.
