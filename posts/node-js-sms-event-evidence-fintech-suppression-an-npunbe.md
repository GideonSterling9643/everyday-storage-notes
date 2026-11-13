# Node.js SMS Event Evidence: Fintech Suppression and Callback Retention Across APIs

The bill for a fintech notification pipeline includes delivery attempts, callback processing, retained event bodies, recipient indexes, and the work of explaining why an invalid destination received another message. For a Node.js application using common SMS APIs, keep a compact decision record for each callback and consult a recipient suppression record before authorizing another notification. Retaining every callback body indefinitely is a separate decision, with a separate evidentiary benefit.

TL;DR: Treat raw payload retention and recipient suppression as different lifecycles. Preserve the evidence supporting a suppression decision, discard redundant payload copies under an explicit retention policy, and never translate an ambiguous delivery failure into permanent recipient invalidity. Email bounces and SMS failures need channel-specific interpretations.

Consider a planning example, not a measured workload: 1,000,000 callbacks per day, an average retained body of 2,000 bytes, and a 90-day window produce 180,000,000,000 bytes of raw bodies before indexes, replicas, or backups. At an assumed 200 bytes per compact decision record, the corresponding record set is 18,000,000,000 bytes. Under those assumptions, raw bodies dominate retained logical bytes. That calculation does not establish which item dominates the actual invoice; delivery charges, storage policies, and operational labor still need local measurements.

Bytes aren't evidence.

## How should Node.js retain SMS event notifications as suppression evidence?

Start with the question an internal review should be able to answer: which destination was blocked, for which channel and purpose, on what evidence, and under which interpretation of that evidence? A callback body can support the answer, but a bucket of bodies cannot substitute for an explicit decision history. Conversely, a compact record that says only “invalid” throws away the reasoning an investigator needs.

For a proposed fintech design, use separate fields for the internal recipient identifier, channel, notification purpose, application message identifier, external message identifier, source event identifier, receipt time, normalized outcome, reason, and adapter version. Keep the suppression decision linked to that record. The adapter version matters because a later mapping change should not silently rewrite the explanation for an earlier block; a corrected interpretation should produce another recorded decision, with its own justification.

A payment receipt addressed to an invalid phone number is the running example here. “Invalid phone destination” should block subsequent SMS attempts within the scope of the suppression policy. A temporary delivery failure should retain its uncertainty. Neither outcome establishes that the customer's email address is invalid, and neither proves that the customer read the receipt. Keep those conclusions separate.

The evidentiary boundary is narrow.

This design supplies an inspectable application history; it does not claim to satisfy a jurisdiction's retention requirements, establish legal consent, or prove customer acknowledgment. Those requirements must determine retention and access policy before deployment.

## Reduce retained bodies without erasing the reason

The change that moves the example's dominant storage term is to shorten raw-body retention while keeping compact decision records for the approved evidence window. If the planning window for raw bodies becomes 7 days, their logical size becomes 14,000,000,000 bytes at the same assumed traffic and body size. Keeping 90 days of the assumed compact records adds 18,000,000,000 bytes. These are capacity calculations, with no claim about compression, billing units, or production performance.

There is a price in diagnostic detail. Decide which evidence survives before deleting anything.

| Retained material | What it lets a reviewer examine | What it cannot establish alone |
| --- | --- | --- |
| Raw callback body | Original fields available to the adapter | Whether the application applied the right policy |
| Compact decision record | Recorded outcome, reason, identifiers, and adapter version | Fields deliberately omitted during normalization |
| Current suppression state | Whether a new send is presently blocked | The full sequence of decisions that created that state |
| Decision history | How and why suppression changed | Missing original payload details after their retention window |

Do not let deletion of old evidence reactivate a destination. In this design, the current suppression state remains until an authorized policy transition changes it; expiration of a raw callback is not such a transition. The opposite mistake is also costly: retaining the current blocked flag forever while dropping every explanation leaves the team unable to distinguish an intentional restriction from a mapping error.

For delivery attempts, estimate a different term: authorized attempts multiplied by the applicable delivery charge. Deduplicating decisions can reduce repeated processing, but it does not retroactively cancel sends. Avoid presenting storage arithmetic as a claim about the total notification budget.

## Connect Node.js SMS callbacks to a channel-specific decision

In a Node.js implementation, the callback handler should authenticate the incoming request according to the selected API's documented contract, resolve the destination through the application's message record, and pass a normalized event into a transaction that records the decision and updates suppression together. Authentication details, acknowledgment rules, and callback fields are API-specific; no universal signature header or callback schema is assumed here. The common interface belongs inside the application.

The Python example below specifies that internal decision function. It is an executable reference for the behavior a Node.js handler should call or implement, rather than an SDK example with invented external fields. Its input values are application-defined outcomes. The adapter must map documented external events into them and preserve an unknown outcome when the available evidence does not justify a stronger conclusion.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Event:
    recipient_id: str
    channel: str
    outcome: str


@dataclass(frozen=True)
class Decision:
    action: str
    reason: str


def decide(event: Event) -> Decision:
    if event.channel not in {"sms", "email"}:
        raise ValueError("Unsupported channel")
    if event.outcome == "invalid_destination":
        return Decision("suppress", "destination_invalid")
    if event.outcome == "temporary_failure":
        return Decision("preserve", "temporary_failure")
    if event.outcome == "delivered":
        return Decision("preserve", "delivery_reported")
    return Decision("preserve", "unclassified_outcome")


example = Event("recipient_257", "sms", "invalid_destination")
assert decide(example) == Decision("suppress", "destination_invalid")
assert decide(Event("recipient_257", "sms", "delivered")).action == "preserve"
assert decide(Event("recipient_257", "email", "unknown")).action == "preserve"
```

Here, `preserve` means leave existing suppression unchanged.

It does not mean authorize a send. That distinction prevents an older delivery report from undoing a later invalid-destination decision, while allowing the application to record both events for review. Reactivation needs its own policy and evidence.

The surrounding persistence layer carries the harder guarantees. Define a source-event deduplication key from the actual API contract, include the source account boundary, and enforce uniqueness in durable storage; a process-local set is insufficient for a design that must survive process restarts. If the source supplies no suitable event identifier, document the limitations of any derived key before treating it as equivalent. Store the decision history entry and suppression transition atomically. A handler should acknowledge durable acceptance only at the point specified by its integration contract, with a recovery path for work accepted but not yet applied.

Email needs another boundary. RFC 7208 defines SPF authorization checks for hosts using a domain in SMTP identities. It distinguishes results including `fail`, `softfail`, `temperror`, and `permerror`; those results must not be relabeled as proof that a recipient mailbox does not exist. A sender authorization failure and an invalid recipient are different findings. SPF supplies no SMS failure taxonomy, so this example deliberately derives none from it.

## Test the evidence lifecycle before changing retention

Replay a duplicate callback, reverse the arrival order of a delivery report and an invalid-destination report, and interrupt processing between receipt and persistence. The acceptance criterion is concrete: repeated input must not create repeated suppression transitions, interrupted work must remain recoverable, and delivery reports must not silently reactivate a blocked destination. Also test the send path. A correct callback processor is insufficient if a queued payment receipt can bypass the suppression check when it is finally authorized for dispatch.

There is a concurrency boundary to specify: this design promises to block future authorizations after suppression commits. It does not claim to recall an attempt already authorized or accepted by an external delivery system.

Record the authorization decision.

Without it, a reviewer cannot distinguish that race from a missing check. Consider two workers handling the same payment receipt: one records destination invalidity while another authorizes a send against an earlier state. A history containing only the eventual blocked flag conceals the ordering that explains the attempt. The proposed evidence record should therefore identify the authorization, its policy version, and its relationship to the message, while the dispatch design must define how concurrent suppression changes and authorizations are serialized. Otherwise the team has specified a useful callback reducer but left its most consequential boundary implicit.

Deploy a new adapter mapping first in comparison mode, recording proposed decisions without changing current suppression. Review disagreement cases, especially unknown outcomes and temporary failures, before enabling writes. Observe duplicate events, unclassified outcomes, processing delay, blocked send authorizations, and suppression changes without linked evidence. Keep recipient identifiers out of metric labels; event-level investigations belong in access-controlled records.

The deliberate stopping point is full callback-body retention beyond the selected diagnostic window. Keep the approved decision history and current suppression state, but stop retaining redundant bodies and unrelated message content merely because storage is available. When an old adapter interpretation is challenged, this choice can leave the team unable to replay every original field. Extending raw retention buys that diagnostic option at the cost of more retained sensitive material and more deletion work. Make that trade explicit before the first expiration job runs.

## References

RFC 7208 is the standards reference for the SPF boundary discussed above. The workload sizes and retention windows are explicitly hypothetical planning inputs; the event schema and decision function specify the proposed application policy, not an external API contract.

## Further reading

- RFC 7208: Sender Policy Framework (SPF) for Authorizing Use of Domains in Email, Version 1 — https://datatracker.ietf.org/doc/html/rfc7208
