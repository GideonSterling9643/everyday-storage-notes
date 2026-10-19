# Edtech Event Notification Stack — 6 Evidence Tests Before Choosing One Provider

Short answer: choose one email-and-SMS operator for a startup's seller order alerts only if a six-test replay can reconstruct the same compliance evidence as a split-vendor design. The tests are trigger, policy, content, dispatch, outcome, and erasure. If either channel fails one test, split that channel behind an application-owned boundary; if both pass, consolidation is easier to operate and is the defensible default.

This decision is about evidence custody, not the lowest message price. An edtech marketplace may need to explain why a seller received an alert for a new workbook order, which destination was selected, what content revision was rendered, and what the delivery system later reported. A convenient SDK cannot answer those questions by itself. **The architecture passes only when retained application records can replay the claim without treating a provider dashboard as the source of truth.**

## Should a startup event notification stack use one provider?

Run 6 tests before comparing contracts. Take one synthetic order, assign it a stable event identifier, and ask each candidate topology to produce an exportable timeline. Then remove access to its live dashboards and repeat the reconstruction from retained records. The exercise is intentionally adversarial: a compliance record that exists only behind a mutable user interface is an operational clue, not durable evidence under the application's control.

The six tests are uneven because their failure costs are uneven. Trigger replay must connect the notification to the committed order event without creating a second logical alert. Policy replay must show the versioned rule that selected email, SMS, both, or neither. Content replay must identify the exact template revision and a digest of canonical rendered content, while avoiding unnecessary copies of addresses, phone numbers, and order details. Dispatch replay must correlate every attempt and retry. Outcome replay must preserve the raw report or a governed reference alongside the normalized state. Erasure replay must demonstrate that personal data can follow its retention policy without destroying the non-personal linkage needed to interpret the timeline.

One distinction prevents a large category error. DKIM gives email a mechanism for a signing domain to claim responsibility for a message, and RFC 6376 explicitly says that verifiers can use that identity as an input to local policy; it does not prove that a human read the message. An operator's acceptance response is narrower still. Record observed states such as `accepted`, `delivery_reported`, `failed`, and `unknown`, and never silently translate them into “seller notified.”

No inference fills the gap.

## Compare evidence custody before channel features

The table compares two sourcing shapes, not named products. A single operator may still expose different semantics for email and SMS, while two specialists may share upstream infrastructure or downstream carriers. Count boundaries you can test, not logos on invoices.

| Replay surface | One operator for both channels | Separate channel operators | Reject the design when |
|---|---|---|---|
| Trigger and policy | Fewer account boundaries, but consolidation does not create application policy evidence | Routing is explicit, although policy can drift between adapters | A retry can select a different channel without a new recorded decision |
| Rendered content | One template system may reduce revisions to reconcile | Channel-specific rendering can remain independently controlled | The exact revision and canonical digest cannot be recovered |
| Dispatch correlation | Fewer external identifier formats | More identifiers and callback schemas | An ambiguous timeout can create an uncorrelated second attempt |
| Outcome history | One export surface, with a larger correlated gap if export is incomplete | More reconciliation work, with narrower replacement scope | Raw reports are overwritten by the latest normalized status |
| Data lifecycle | One retention review may cover both channels | Different stores can honor different retention requirements | Deletion of personal data makes the remaining evidence unintelligible |
| Operational ownership | Fewer credentials, rotations, and escalation paths | Channel failures can be handled and migrated independently | Nobody owns stale callbacks, export checks, and replay drills |

The one-operator shape wins when a small team can prove all six rows with fewer credentials, callback verifiers, reconciliation jobs, and retention reviews. Its limitation is correlated control-plane and account risk: it is not suitable when an independently operable channel is a written requirement, or when either channel's evidence cannot be exported. The split shape wins when a channel has a concrete evidence requirement the consolidated interface cannot express or export, or when independent migration and failure boundaries justify the added operations. Its downside is a larger reconciliation surface, so it is not suitable when the team cannot continuously own two callback mappings, two credential lifecycles, and two export checks. This is the trade-off. Neither shape is inherently cheaper once engineering ownership, export validation, and incident investigation enter the model, so volatile per-message prices should remain a procurement input rather than the architecture's main argument. There is a hard limit here: evidence portability is meaningful only if it is exercised. Schedule an export-and-replay test after schema changes and credential rotations. A dormant export checkbox is a promise; a fixture that rebuilds the timeline is evidence.

## Put the replay contract on the critical path

The order transaction should hand off notification work durably, using an outbox or an equivalent atomic mechanism, before any network dispatch. The evidence store remains application-owned. Adapters translate requests and reports, but they do not decide purpose, eligibility, or whether a second channel is permitted.

The Python sketch below encodes that narrow contract. It deliberately stores destination references instead of raw contact details and requires content to be canonicalized before hashing with SHA-256; hashing arbitrary serialization would produce unstable evidence. In production, `append` and `enqueue` must participate in the storage transaction that makes the notification plan durable.

```python
from dataclasses import dataclass
from hashlib import sha256
from json import dumps
from typing import Literal, Protocol

Channel = Literal["email", "sms"]


class EvidenceStore(Protocol):
    def append(self, record_id: str, record: dict) -> None: ...


class DispatchOutbox(Protocol):
    def enqueue(self, attempt_id: str, channel: Channel, body: dict) -> None: ...


@dataclass(frozen=True)
class Decision:
    policy_revision: str
    channels: tuple[Channel, ...]
    destination_refs: dict[Channel, str]
    template_revisions: dict[Channel, str]


def content_digest(rendered: dict) -> str:
    canonical = dumps(rendered, sort_keys=True, separators=(",", ":"))
    return sha256(canonical.encode("utf-8")).hexdigest()


def plan_seller_order_alert(
    evidence: EvidenceStore,
    outbox: DispatchOutbox,
    order_event_id: str,
    seller_ref: str,
    decision: Decision,
    rendered: dict[Channel, dict],
) -> None:
    notification_id = f"seller-order:{order_event_id}:{decision.policy_revision}"
    evidence.append(notification_id, {
        "kind": "notification_plan",
        "order_event_id": order_event_id,
        "seller_ref": seller_ref,
        "policy_revision": decision.policy_revision,
        "channels": decision.channels,
        "template_revisions": decision.template_revisions,
        "content_digests": {
            channel: content_digest(rendered[channel])
            for channel in decision.channels
        },
    })

    for channel in decision.channels:
        attempt_id = f"{notification_id}:{channel}:1"
        evidence.append(attempt_id, {
            "kind": "dispatch_planned",
            "notification_id": notification_id,
            "channel": channel,
            "destination_ref": decision.destination_refs[channel],
        })
        outbox.enqueue(attempt_id, channel, rendered[channel])


def append_delivery_report(
    evidence: EvidenceStore,
    report_id: str,
    attempt_id: str,
    observed_state: str,
    occurred_at: str,
    raw_report_ref: str,
) -> None:
    evidence.append(report_id, {
        "kind": "delivery_report",
        "attempt_id": attempt_id,
        "observed_state": observed_state,
        "occurred_at": occurred_at,
        "raw_report_ref": raw_report_ref,
    })
```

Callbacks can arrive twice, late, or in an order that contradicts the application's preferred state progression. Append them under stable report identifiers and retain the raw form or governed reference; do not update one “current status” row and discard the path taken. An ambiguous request timeout is also a named failure mode: retry with the same attempt identity where the operator contract supports that behavior, otherwise reconcile before creating a new attempt.

If an AI agent chooses a channel or invokes the dispatch adapter, its tool schema should constrain the allowed fields and channel values, and the application must validate the call before execution. Tool-use documentation describes the model as requesting a tool while the client application executes it; that separation is useful here because the model's request is not delivery evidence. The policy decision, validated arguments, adapter attempt, and later report remain distinct records.

## Failure drills define the real boundary

Start with duplicate and missing events. Replay the same order event twice and verify that it produces one logical notification plan. Deliver the same outcome report twice. Delay the outcome until after a worker restart. Deny writes to the evidence store while the outbox is available, then reverse the failure. A correct design either commits the plan and durable handoff together or commits neither; it must not send first and invent evidence afterward.

Next, exercise disagreement. Feed `delivery_reported` and then a later `failed` report for the same attempt. The normalized view may apply a documented projection rule, but the append-only history must retain both observations and their event times. Rotate one adapter credential and confirm that queued work retains its attempt identity. Remove personal destination data according to policy, then verify that an authorized reviewer can still connect the remaining pseudonymous reference, content digest, dispatch attempt, and outcome without recovering the deleted value.

Measure gaps rather than celebrating sends: notification plans without dispatch attempts, attempts without recent reports, rejected callback signatures, unknown raw event shapes, replay failures, and the age of the oldest outbox item. The alert thresholds depend on the marketplace's service objective and cannot be inferred from a vendor's marketing page. Define them in the operating agreement, test them with controlled failures, and version them beside the adapter mapping.

Six fixtures are the minimum shape of this decision record, one for each replay test. They aren't a claim about adequate test coverage. Add fixtures for every accepted raw report shape and every policy transition the application permits, because a mapping change that cannot replay old evidence is a data migration even when no database column changes.

## The rejected option still has a valid use case

This record rejects selecting separate email and SMS operators merely to obtain theoretically independent channels. The extra credentials, callback verification rules, exports, schema mappings, retention reviews, and on-call paths are real failure surfaces, while independence is unproven until common dependencies and application bottlenecks are tested. For a startup sending ordinary new-order alerts, that overhead is unjustified when one operator passes all six replay tests.

Split sourcing remains valid when a documented channel requirement fails the consolidated replay: an outcome field cannot be exported, retention cannot be configured to the required policy, regional processing constraints differ, or a channel must be migrated without disturbing the other. **Split at the adapter boundary only after a failing fixture makes the reason reproducible.** That preserves the decision rule during procurement changes and keeps “easy integration” from becoming permanent evidence lock-in.

The final acceptance review is short. Hand a synthetic seller-order identifier and an offline export to an engineer who did not build the integration. If they can reconstruct the trigger, policy revision, content digest, destination reference, attempts, reports, and permitted erasure without a live operator account, the topology is defensible. If they cannot, adding another vendor will not repair the missing record model.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Anthropic, Tool use overview: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
