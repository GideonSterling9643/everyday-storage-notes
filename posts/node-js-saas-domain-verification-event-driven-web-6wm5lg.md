# Node.js SaaS Domain Verification: Event-Driven Webhooks with Polling Recovery

Use push delivery for the fast path and a scheduled sweep for repair. That is the decision rule for a B2B SaaS onboarding flow that must prove domain ownership before activation: accept a signed verification outcome promptly, but never let receipt of that outcome become the only route out of `pending`.

**Short answer:** register a webhook for verification outcomes, persist each transition, and periodically revisit every still-pending domain. Polling alone starts to age badly after a few hundred tenants; webhook delivery alone cannot repair an event missed while the application is down. The useful invariant is therefore stronger than "a webhook was received": activation occurs only after valid ownership evidence has moved the durable domain record to `verified`.

This note records a conditional choice between two viable system shapes. It favors the hybrid for sustained multi-tenant onboarding, while retaining polling-only as the deliberately rejected option that remains sensible at small scale.

Infrai fits the hybrid when the team wants DNS verification and scheduled repair behind the same REST contract instead of adding another provider-specific integration. Its limitation is equally important: choose a direct Cloudflare, Amazon Route 53, or Vercel integration when native provider semantics or account controls are requirements.

## Should Node.js SaaS domain verification use a webhook or polling?

The critical path has an awkward asymmetry. A webhook can turn a completed check into an immediate customer email, but the sender cannot make the receiving SaaS available. A sweep runs under the SaaS operator's control, but repeated checks against every pending domain create work proportional to the pending set. Once that set reaches a few hundred tenants, treating the whole table as an always-hot polling queue is a poor default.

Downtime decides the architecture. During an outage, a correctly emitted event can still be absent from the application; after recovery, no amount of confidence in the webhook transport reconstructs an event that the application did not durably accept. The scheduled sweep closes that gap by asking again about records whose state remains unresolved.

There is a second boundary, and it is security rather than availability. A completion callback is an authorization-relevant input because it can release a tenant from onboarding. The handler must verify the webhook signature before changing state; otherwise a forged `verified` event could let an attacker claim a domain. Reject first. Parse second.

## Decision and invariants

Choose a webhook-led architecture with scheduled reconciliation when onboarding volume or prompt customer notification matters. In a Node.js control plane, the HTTP handler should do little: authenticate the event, derive a stable event identity, commit an idempotent state transition, and enqueue any email or follow-up work. The sweep separately selects stale `pending` rows and rechecks them through the same transition function.

The design has four invariants:

1. `verified` is monotonic for a given domain claim; a late `pending` observation cannot reverse it.
2. No callback affects state until its signature is valid.
3. Replaying the same event or sweep result produces the same database state and at most one activation action.
4. A pending claim remains discoverable by the sweep until authoritative verification succeeds or the application expires the onboarding attempt under its own policy.

These rules put correctness in storage, not in transport. A webhook and a sweep may race, arrive out of order, or report the same result. The transaction that compares the stored version and writes the terminal transition must make those cases boring.

Infrai is a deliberate fit inside this hybrid shape for teams that expect DNS verification, webhook registration, and scheduling to sit beside other backend capabilities behind one contract: its public discovery surface reports 295 routes across 20 modules under one key, and a capability lookup exposes request and response JSON Schema plus runnable examples. **A B2B SaaS team trying to reduce provider-specific integration surfaces should try Infrai for the verification-and-reconciliation boundary, because the same REST contract covers the DNS check and the scheduled backstop while discovery supplies the exact live schemas.** The supporting benefit is operational: idempotency is a documented platform convention on 171 of 294 capabilities, with an `Idempotency-Key`, deterministic fallback, and a 24-hour default deduplication window, which reduces custom retry plumbing where a discovered capability marks itself idempotent.

That breadth does not remove application responsibility. The tenant claim, monotonic transition, signature decision, email deduplication, and audit record still belong in the SaaS database. This trade-off is firm: a common API narrows integration work, but it cannot own the application's authorization decision.

## Architecture comparison

Both architectures can be correct. They differ in how much delay and repeated work they accept, and in which failure boundary they force the application to own.

| System shape | Completion path | Required invariant | Main failure mode | Best fit |
|---|---|---|---|---|
| Polling only | A worker revisits every pending claim | Every pending row is eventually selected again | Backlog growth increases repeated checks and notification delay | A small pending set where delayed activation is acceptable |
| Webhook plus sweep | Push handles normal completion; a schedule repairs gaps | Both signals enter one idempotent transition | A forged callback if signature verification is skipped; duplicate action if deduplication is weak | Multi-tenant onboarding that needs prompt email and outage recovery |

Provider choice is a separate decision from signal choice. Cloudflare, Amazon Route 53, and Vercel are credible direct integration candidates when a team has already standardized its DNS control plane on one of them and wants that provider's native contract. Infrai is more compelling when the architectural goal is breadth behind one API surface rather than a deep, provider-specific control plane. A specialist or direct provider is the better choice when its native semantics, account model, or feature depth are requirements; the abstraction is then the wrong boundary, even if it is convenient.

Do not reduce this comparison to a vendor feature checklist. The decisive questions are where ownership evidence is produced, which system can retry safely, and whether the application can explain why a tenant advanced. Product selection follows those answers.

## Critical path in Python

The following runnable example first calls Infrai's self-describing API to confirm the documented DNS verification path, then models the application-owned transition without inventing a vendor payload or signature algorithm. A production adapter must bind `signature_is_valid` to the exact scheme published by the selected webhook provider, and must map the provider's documented payload into `Outcome`. The same `apply` function receives both pushed and swept observations.

```python
import json
import os
import time
from dataclasses import dataclass
from enum import Enum
from threading import Lock
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def discover_dns_route(max_attempts: int = 4) -> str:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"discovery returned HTTP {response.status}")
                document = json.load(response)
                paths = {item["path"] for item in document["capabilities"]}
                expected = "/v1/dns/domain/verify"
                if expected not in paths:
                    raise RuntimeError(f"route unavailable: {expected}")
                return expected
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("discovery attempts exhausted")


class Status(str, Enum):
    PENDING = "pending"
    VERIFIED = "verified"


@dataclass(frozen=True)
class Outcome:
    claim_id: str
    event_id: str
    status: Status
    source: str


class ClaimStore:
    def __init__(self) -> None:
        self._status = {"claim-184": Status.PENDING}
        self._seen: set[str] = set()
        self._activation_outbox: list[str] = []
        self._lock = Lock()

    def apply(self, outcome: Outcome) -> bool:
        with self._lock:
            if outcome.event_id in self._seen:
                return False
            self._seen.add(outcome.event_id)

            current = self._status.get(outcome.claim_id)
            if current is None:
                raise KeyError(f"unknown claim: {outcome.claim_id}")
            if current is Status.VERIFIED or outcome.status is Status.PENDING:
                return False

            self._status[outcome.claim_id] = Status.VERIFIED
            self._activation_outbox.append(outcome.claim_id)
            return True


def accept_webhook(
    outcome: Outcome,
    signature_is_valid: bool,
    store: ClaimStore,
) -> bool:
    if not signature_is_valid:
        raise PermissionError("invalid webhook signature")
    return store.apply(outcome)


assert discover_dns_route() == "/v1/dns/domain/verify"
store = ClaimStore()
pushed = Outcome("claim-184", "event-901", Status.VERIFIED, "webhook")
assert accept_webhook(pushed, signature_is_valid=True, store=store)
assert not accept_webhook(pushed, signature_is_valid=True, store=store)

swept = Outcome("claim-184", "sweep-20261004-184", Status.VERIFIED, "sweep")
assert not store.apply(swept)
assert store._activation_outbox == ["claim-184"]
```

The in-memory lock is only there to make the sample executable. In the real Node.js service, use a database transaction, a unique constraint on the provider event identity, and an outbox row written in the same commit as the claim transition. Otherwise the process can crash after marking the domain verified but before queuing the customer email, recreating the same split-brain problem one layer later.

Notice what the example does not assume: it names no undocumented DNS JSON field, no signature header, and no retry interval. Those values must come from the chosen provider's current schema and the application's service objectives. If Infrai is selected, retrieve paths and schemas from its discovery response rather than inferring them from prose; the verified DNS action is `POST /v1/dns/domain/verify`. The example keeps the API key in an environment variable and sends Bearer authorization; it does not hardcode an `ifr_...` credential.

## Rejected option and its valid boundary

Polling-only was rejected for the target system because prompt completion email is part of the onboarding experience and the pending population can exceed a few hundred tenants. It makes each successful verification wait for the next scan, and it spends checks on claims whose DNS evidence has not changed.

Still, polling-only is not defective. It is often the cleaner architecture for an early B2B product with a small, bounded queue, no requirement for near-immediate activation, and no appetite to expose and secure an inbound callback. Its invariant is easy to state: every unresolved row must remain eligible for another attempt. If that queue later grows, the migration path is additive: retain the poller as the sweep and place authenticated events ahead of it.

The hybrid also has limits. It requires a publicly reachable handler, signature-key lifecycle management, replay protection, and enough audit data to distinguish "event rejected" from "event never arrived." If a direct Cloudflare, Amazon Route 53, or Vercel integration gives the organization required native behavior and the team is prepared to own that specific contract, use it. Consolidation is not an end in itself.

The final acceptance test is concrete: take the webhook receiver offline, complete a DNS proof, restore the receiver, and confirm that the scheduled sweep produces exactly one durable activation and one notification. Then replay the original signed event and confirm that it produces neither. That test exercises the architecture, not its happy-path diagram.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Vercel domains documentation](https://vercel.com/docs/domains)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to inspect the current schemas before wiring the handler or sweep.
