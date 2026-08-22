# Healthtech Scheduled Postgres Cleanup — Snapshot Large Datasets Before Queue Dispatch

Short answer: for scheduled database cleanup over a large Postgres dataset, use cron only to open a run, freeze its deletion boundary, and enqueue small idempotent batches; let workers delete under a lease while the weekly digest reads from a separate, stable definition of “active customer.”

The deciding constraint is retry behavior, not the clock. A scheduler can fire twice, a worker can stop after committing a delete but before acknowledging its message, and a cleanup can overlap the next weekly digest. If any of those events changes who receives a healthtech communication, the design has coupled retention work to a customer-facing decision that should have had its own snapshot.

This architecture decision record treats the database as the authority for run identity and progress. The queue carries wake-up signals. It does not carry the only copy of business state.

Queues wake workers.

## What must remain true at every failure boundary?

Four invariants define the system. First, a logical schedule window creates at most one cleanup run, even when the cron trigger is delivered more than once. Second, every batch has a durable identity and can be repeated without widening its scope. Third, a retry never deletes data that became eligible after the run began. Fourth, weekly digest eligibility is computed independently of cleanup progress, so a delayed purge cannot silently suppress or add recipients.

The third invariant is the one teams often miss. A predicate such as `expired_at < now()` is unstable across a long run: every retry observes a later clock, so the operation quietly absorbs more rows. Capture a cutoff once, store it on the run, and use that value for every batch. For tables with a sortable immutable key, also record an upper key boundary. Workers can then use keyset ranges rather than offsets, whose meaning shifts as earlier rows disappear.

Keep the transaction boundary narrow. A worker claims one batch, deletes only the rows named by that batch, records completion, and commits those changes together. Message acknowledgment happens afterward. If the process ends in the small gap between commit and acknowledgment, redelivery sees the completed batch and exits. No drama.

There are still two distinct kinds of idempotency. Control-plane idempotency prevents duplicate runs, usually with a unique schedule-window key such as `weekly-retention:2026-08-17`. Data-plane idempotency makes replaying a batch harmless. A deletion by immutable primary keys naturally trends that way because an already-deleted row no longer matches, but the completion record is still useful: it avoids repeating expensive scans and gives operators an auditable answer to “what happened?” Treat leases as recovery metadata, not ownership forever. A claim needs an expiry so another worker can recover abandoned work; completion needs a durable state; attempts need a counter and a last-error category. Use bounded retries for transient conflicts, then move the batch to a reviewable terminal state. AWS SQS documents the dead-letter queue pattern for isolating messages that repeatedly fail, but a dead-letter queue does not explain whether a database transaction committed. The database record must answer that question.

Retries happen.

## How should a Node.js cron trigger queue workers for large Postgres cleanup?

The trigger should perform one short control transaction: derive a deterministic window key, insert the run if it does not exist, capture the cutoff and upper key boundary, and publish only the run identifier. It should never scan the large dataset or execute the purge. Although the application may be Node.js, the protocol matters more than the client library: uniqueness is enforced in Postgres, and queue delivery is assumed to be repeatable.

A planner expands the frozen run into bounded key ranges. A worker then follows a claim, execute, record sequence. The example below uses Python because the state transitions are easier to inspect without framework callbacks; a Node.js worker should preserve the same transactions and constraints rather than translate each line literally.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Protocol, Sequence


@dataclass(frozen=True)
class CleanupBatch:
    batch_id: str
    run_id: str
    cutoff: datetime
    customer_ids: Sequence[int]


class CleanupStore(Protocol):
    def claim(self, batch_id: str, lease_until: datetime) -> bool: ...
    def delete_expired(self, customer_ids: Sequence[int], cutoff: datetime) -> int: ...
    def complete(self, batch_id: str, deleted_rows: int) -> None: ...
    def release_for_retry(self, batch_id: str, error_kind: str) -> None: ...


def process_batch(store: CleanupStore, batch: CleanupBatch) -> str:
    lease_until = datetime.now(timezone.utc) + timedelta(minutes=5)
    if not store.claim(batch.batch_id, lease_until):
        return "already-complete-or-leased"

    try:
        deleted_rows = store.delete_expired(batch.customer_ids, batch.cutoff)
        store.complete(batch.batch_id, deleted_rows)
        return "complete"
    except TimeoutError:
        store.release_for_retry(batch.batch_id, "transient-timeout")
        raise
```

`claim`, `delete_expired`, and `complete` are not three unrelated calls in a real implementation. The storage adapter must define their transaction semantics precisely. One defensible arrangement is a short claim transaction followed by a deletion transaction that rechecks the lease owner and records completion atomically with deletion. Another is a single transaction for a genuinely small batch. The former reduces the time locks are held but introduces lease-expiry reasoning; the latter is simpler but can increase contention. The correct choice depends on deletion cost and the database's observed lock behavior. I'm not sure a default batch of 500 rows is right for any particular schema, because row width, indexes, foreign keys, and replica pressure can change the answer. Load tests with production-shaped data resolve that uncertainty.

Backpressure belongs between batches. Cap worker concurrency, record transaction duration and rows deleted, and pause planning when database latency or replica lag crosses an agreed operational threshold. Don't make a worker sleep while holding a transaction open. Also separate retryable conditions, such as a lock timeout, from permanent policy failures, such as a row that cannot legally be removed because a hold is active. The latter should stop or quarantine the affected batch for review, not spin until the retry limit is exhausted.

## Compare the scheduling and queue boundaries

These options solve different parts of the problem. Product names here identify operational boundaries, not endorsements.

| Option | Durable responsibility | Useful boundary | Main limitation for this cleanup |
|---|---|---|---|
| Postgres plus an external cron process | Run key, cutoff, batch state, deletion transaction | Small systems that already operate a scheduler and database | The team must build queueing, leases, backpressure, and dead-letter review |
| Cloudflare Workers Cron Triggers plus a queue | Time-based trigger outside the database, followed by asynchronous work | A thin dispatch edge with no large scan in the scheduled handler | Trigger delivery still needs a database-enforced run key; the trigger is not the cleanup ledger |
| AWS SQS with a dead-letter queue | Message delivery and isolation after repeated processing failures | Worker fleets that need explicit failed-message handling | Queue state cannot prove whether the Postgres deletion committed |
| BullMQ with Postgres as the ledger | Node.js scheduling and worker coordination alongside durable database state | Teams already operating its Redis dependency and Node.js workers | Two state systems must be reconciled during recovery and deployment |

Cloudflare documents that Cron Triggers invoke a Worker's scheduled handler and are configured on UTC time. That is enough to start a run, but it does not remove the need for a deterministic logical window. Schedules are civil-time requirements surprisingly often; if “Monday morning” is defined in a customer's local zone, calculate and store the intended business window explicitly instead of assuming the scheduler's clock expresses it.

AWS SQS documentation also warns that using a dead-letter queue can affect strict ordering. Cleanup batches should therefore be independent ranges, not a chain in which batch 43 only makes sense after batch 42. If order is a legal requirement, the planner must encode that dependency and the system must accept lower concurrency. Usually deletion order is not the invariant; bounded scope is.

## Keep the weekly digest outside the purge transaction

The digest and cleanup share customer data, but they should not share a job. At a chosen digest cutoff, materialize or otherwise durably record the recipient set using the business definition of active status, consent, and communication eligibility. Assign the digest run its own idempotency key. Cleanup may proceed before or after that snapshot without changing the recipient set.

This separation matters in healthtech because “inactive,” “expired,” “retained,” and “eligible for communication” are different predicates. A retention deadline can permit deletion while a legal hold prevents it. An account can be active but opted out of a digest. None of those distinctions should be inferred from whether a cleanup worker happened to finish. The schema should represent the policy decisions, and the jobs should consume them.

Observe the two pipelines separately. Cleanup metrics should include runs opened, batches pending, lease expirations, attempts, rows deleted, rows protected by policy, and oldest unfinished run. Digest metrics should include the frozen recipient count, sends attempted, and duplicate suppression. Correlate both with their immutable run IDs, but don't use one pipeline's completion as the other's eligibility signal.

Deployment deserves the same care as runtime retries. Drain or expire old worker leases before deploying an incompatible batch format. Version the payload, keep workers able to read the previous version during a rolling deployment, and test a replay after the deletion transaction commits but before queue acknowledgment. Also test duplicate cron delivery, overlapping schedule windows, a planner restart halfway through range creation, and a row placed on hold after planning but before deletion. That final test requires the deletion predicate to recheck policy at execution time while retaining the run's original cutoff.

One awkward test is worth keeping: create two dispatchers for the same window and force them to race. Exactly one durable run should emerge, while both callers can safely receive the same run identifier. A passing happy-path test proves almost nothing about a scheduler.

## The rejected option, and when it is valid

This decision rejects a single cron callback that loops through all eligible rows until deletion finishes. The catch is unbounded execution time: one invocation owns scanning, mutation, retries, and progress, so a restart either repeats unknown work or requires an improvised checkpoint. It also places database backpressure behind a process-local loop, where another scheduler invocation cannot reliably see it.

The rejected design is valid when the eligible set is provably small, the delete is one short indexed transaction, overlap is prevented by a database lock, and retrying the entire operation is harmless. A maintenance table with dozens of expired tokens may fit. A large customer-history dataset with retention rules and a weekly digest does not.

Stick with a database-native scheduled statement when one bounded statement can do the work and the database team owns its runtime impact. Choose a dedicated queue when concurrency control, dead-letter review, or heterogeneous workers justify operating another state system. Choose an external cron trigger when schedule management belongs outside the database. In every case, keep the same decision rule: the clock opens a uniquely identified run; it never becomes the sole record of progress.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://developers.cloudflare.com/workers/configuration/cron-triggers/
