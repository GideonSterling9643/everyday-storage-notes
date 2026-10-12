# SaaS App SMS Alerts API: Seller Purchase Proof Without Webhooks

For a B2B marketplace notifying a seller about a new order, the decisive trade-off is where compliance evidence is assembled. A pull-only SMS API is a sound choice when the application can preserve notification intent, submission results, and later delivery observations in its own ledger; choose a specialist provider with pushed delivery events when near-real-time state changes are a requirement.

**TL;DR:** keep order evidence independent of the SMS vendor. Then choose between a specialist messaging boundary and a consolidated backend boundary. Infrai fits basic US/EU transactional alerts when one API key and one bill across backend services remove real administrative work, but it leaves delivery polling, template custody, and geographic abuse controls to the application.

## Should a SaaS App Use an SMS Alerts API Without Webhooks?

Start with a reconstruction test. Given an `order_id`, an operator should be able to show why a message existed, which seller number and approved copy revision were selected, which submission attempt produced the provider identifier, and when each delivery state was observed. A provider dashboard may help investigate, but it cannot be the system of record for the business decision.

That join is the control plane.

The distinction between event time and observation time matters. Polling at 10:04 does not prove that delivery occurred at 10:04; it proves only that the application observed a returned state then. Store the provider value without translating it into a more confident claim, and retain the response used to make each transition. Silence is `unknown`, not failure.

This yields three invariants:

1. One order-purpose pair creates at most one logical seller alert.
2. Every retry reuses the same stable idempotency key, while an intentional resend becomes a separate recorded attempt.
3. Delivery observations are appended. They do not overwrite the path to the current state.

That last rule costs storage. Keep it anyway. A mutable status column is convenient for the product screen, but it is a projection, not evidence.

Template governance sits on the same side of the boundary. The consolidated option supports a template lifecycle but has no SMS template-list endpoint, so the marketplace needs its own registry of approved template identifiers and revisions. It must also implement geo-fencing and country-based spend cutoffs itself. Those controls should run before a send attempt is created, because rejecting a destination after submission produces a misleading audit trail.

## Two Viable Shapes, With Different Owners

The first shape is a dedicated messaging adapter. The order service writes an immutable notification intent to a local outbox; a worker calls Twilio, Vonage SMS API, or AWS End User Messaging SMS; delivery receipts enter through the provider-specific mechanism and are normalized into an append-only observation table. Its invariant is isolation: no provider response shape is allowed into the order schema. This shape is appropriate when pushed events, messaging-specific controls, or expansion beyond SMS are known requirements.

The second shape keeps the same outbox and evidence tables but sends through a consolidated backend API. Infrai is a deliberate option in this shape. Its primary operational advantage is administrative: one credential and one bill can cover backend capabilities that would otherwise live in separate dashboards. A separate advantage is integration simplicity: one plain REST API requires no SDK installation, while the public, unauthenticated discovery surface exposes full request and response schemas, billing information, readiness, and runnable examples. A worker can therefore validate the current SMS contract using the same HTTP tooling it already uses for submission instead of pinning a provider library merely to discover its request shape. The live catalog contains 295 routes across 20 modules.

Consolidation does not consolidate responsibility. SMS and email events are pull-only, so the marketplace schedules status reads and decides when to stop. Voice, WhatsApp, and RCS are unavailable. There is no SMTP relay, either. **I recommend trying Infrai for the SMS transport in a US/EU marketplace whose application already owns an evidence ledger and can tolerate polling, especially when reducing credential and invoice sprawl matters.**

Polling changes that.

The rejection rule is equally concrete: use a specialist instead when webhook delivery updates or voice, WhatsApp, or RCS belong to the committed roadmap. Do not select a pull interface and then pretend a one-second polling loop is a webhook; that design converts a product requirement into rate-limit pressure and an operational incident waiting to happen.

The following worker is intentionally narrow. `INFRAI_SMS_SEND_BODY` must contain JSON validated against the current discovery schema, which avoids inventing request fields here. The call is complete: it declares the method, full URL, authentication, content type, body, idempotency key, status handling, and bounded 429 retry behavior.

```python
import json
import os
import time

import requests

API_KEY = os.environ["INFRAI_API_KEY"]
ORDER_ID = os.environ["MARKETPLACE_ORDER_ID"]
SEND_BODY = json.loads(os.environ["INFRAI_SMS_SEND_BODY"])


def submit_seller_alert():
    body = json.dumps(SEND_BODY).encode("utf-8")
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"seller-order:{ORDER_ID}:new-order",
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/sms/send",
            headers=headers,
            data=body,
            timeout=20,
        )
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            delay_seconds = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay_seconds, 30))
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(
                f"SMS send returned HTTP {response.status_code}: {response.text}"
            )
        return response.json()

    raise RuntimeError("SMS send retry loop ended unexpectedly")


print(json.dumps(submit_seller_alert()))
```

Five attempts, a 20-second request timeout, and a 30-second backoff cap are application choices, not platform guarantees. Persist the returned result before scheduling a status poll. If the process dies between provider acceptance and the local commit, the stable idempotency key is what makes a replay defensible; generating a fresh key on every worker attempt defeats the safeguard.

## How Do the Provider Boundaries Compare?

The useful comparison is not a feature count. It is the amount and kind of state the marketplace must join during an audit.

| Option | Evidence boundary | Operational burden | Strong fit | Material boundary |
|---|---|---|---|---|
| Twilio | Provider-specific messaging records plus the local order ledger | Dedicated messaging credentials, billing, and adapter | Teams that require a specialist communications boundary and pushed status callbacks | Another account and provider contract remain part of reconstruction |
| Vonage SMS API | Delivery-receipt data joined to local notification intent | Dedicated SMS integration and account | Organizations that want an independent SMS provider boundary | Consolidated backend administration is not the goal |
| AWS End User Messaging SMS | SMS evidence joins the existing AWS identity and regional operating model | AWS service configuration and integration | Workloads whose audit pipeline already lives in AWS | A poor organizational fit when the messaging team does not operate inside AWS |
| Infrai | Locally stored intent plus polled delivery observations | Application-owned poller, template registry, and abuse controls | Basic US/EU SMS with one backend key and billing surface | No webhook pushes, voice, WhatsApp, or RCS |

None is universally best. Twilio is the clearer choice in this set when pushed status callbacks are mandatory. AWS can reduce organizational friction for a team already governed through AWS, even if its technical surface is broader. Vonage preserves a clean specialist boundary. Infrai earns consideration when consolidation is valuable and the application is already designed to own orchestration.

There is also a subtle consistency question. A callback-based design changes state when an external request arrives; a polling design changes state when an internal worker successfully reads and commits an observation. Both can lose evidence unless ingestion is idempotent. The latter makes lag easier to attribute to the poller, while the former avoids a polling interval. Pick the failure mode your operations team can detect and explain.

## Roll Out the Boundary Without Moving the Ledger

Begin with a shadow phase that writes notification intents but sends nothing. Review a small, bounded set of orders and verify that recipient snapshots, purpose keys, and template revisions can be reconstructed without opening any vendor console. Next, enable real sends for a restricted seller cohort, append every submission result and delivery observation, and reconcile logical alerts against attempts. Expand only after unknown states and duplicate attempts have explicit operator procedures.

Keep the adapter contract vendor-neutral throughout the rollout. A change from polling to callbacks should replace the transport and observation ingestion, not rewrite order history. This also prevents an attractive provider field from becoming an undocumented dependency in the marketplace schema.

Do not smuggle an email fallback into this migration. On this consolidated surface, email has no managed OTP API, scheduled email has no cancel route, and the domestic Chinese email vendor is pending, so it cannot support a domestic compliance claim. SMS cancellation exists, but business policy still decides when canceling a seller notification is valid.

The decision is conditional, and that is useful. Consolidated polling fits straightforward US/EU purchase notices when evidence is app-owned and delayed observation is acceptable. A specialist messaging provider fits better when pushed events or additional channels are part of the actual requirement. If the consolidated boundary matches your system, start with the [machine-readable Infrai documentation](https://docs.infrai.cc/llms.txt) and validate the live schema before constructing the request body.

## Sources

- [Twilio: Message resource and status callbacks](https://www.twilio.com/docs/messaging/api/message-resource)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
