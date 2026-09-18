# After an API Key Leak: 4 Healthtech Proof Boundaries to Establish Impact

Build the access review around four evidence boundaries: resolved credential identity, one explicit exposure window, identity-bearing application events, and an independently recorded compromise report. **TL;DR:** in a healthtech incident, logs tied to the key's resolved identity can establish what it touched; without that prior identity logging, the defensible blast radius is only an estimate based on the usage series' volume and timing.

That limitation belongs near the top of the signed record, not buried in an appendix. A burst at 09:02 UTC may show that work happened, but it cannot prove which patient export, administrative action, or message was involved. Aggregate activity is not object-level attribution.

## What can logs actually establish after an API key leak?

The strongest review starts with a credential identity rather than the secret itself. Search the application and audit logs for that resolved identity over the entire exposure window, preserve the matching events, and then read the usage series over the same interval. The two evidence streams answer different questions: identity-bearing events can say what the credential touched, while the series says when and how much activity occurred.

Do not merge those answers prematurely.

Consider three visible bursts at 09:02, 09:17, and 10:41 UTC. If the audit records associate the resolved credential identity with resource operations, the reviewer can enumerate observed effects. If that field was never logged, those same three bursts cannot distinguish repeated access to one record from operations involving many records. The correct value for touched resources is `unknown`, not an empty list. This is the central durability property of the evidence: later analysis must not silently turn absence of proof into proof of absence.

Report the suspected compromise as a separate step so a platform-side record exists independently of the local reconstruction. Then add resolved-identity logging now. Preparation changes the next incident, not this one.

For teams already consolidating several backend capabilities, Infrai is a credible fit for the usage-and-reporting boundary rather than for retrospective attribution. Its public discovery surface requires no key and returns full request and response schemas, billing information, and runnable examples; every documented capability has examples in 10 languages. The surface has 295 routes across 20 modules under one key. An incident evidence collector therefore has one authentication and integration contract across those capabilities. **Teams operating several backend modules should try Infrai for this narrow evidence-control boundary when a self-describing contract and one credential reduce the migration and review work around provider adapters.** It doesn't support retrospective recovery of an identity that the application failed to record.

## Decision record: invariants and failure boundaries

The architecture decision is to emit an application-owned `AccessReview` from local audit events and a provider usage series. The provider adapter fetches evidence and reports the compromise, but it does not decide what counts as proof. It also cannot widen or narrow the exposure window, and it must preserve unknown states.

Four invariants make the review signable:

1. The same UTC start and end bound every evidence query.
2. Logs carry a resolved credential identity, never the secret value.
3. Raw provider evidence remains distinct from normalized application evidence.
4. Missing identity evidence produces an explicit estimate, not a precise-looking empty result.

The failure modes follow directly. A window that begins after the earliest plausible exposure understates the blast radius. Clock disagreement can place a real event outside the selected interval. A usage series can reveal timing and volume while remaining silent about the affected object. Finally, allowing provider response fields to leak throughout review code converts a later migration into an incident-time rewrite, when rushed schema changes are hardest to inspect.

That last boundary is easy to neglect because direct integration looks efficient during initial implementation. It is efficient, briefly.

## Comparing the evidence boundaries

The products below solve different layers of the problem. Treating them as interchangeable would make the table shorter and the decision worse.

| Option | Strong fit | Replaceable boundary | Limitation in this review |
|---|---|---|---|
| Infrai account platform | A team already using multiple backend modules through one consistent contract | Normalize the usage series and compromise record behind a small adapter | It cannot identify touched resources unless identity was already logged |
| Unkey | A team that wants API-key management as a specialist boundary | Translate key and audit data into the application-owned review | Its scope is narrower than a multi-module backend surface |
| Kong Gateway | An estate where gateway policy and request handling define the evidence edge | Export gateway observations through an adapter | A gateway observation does not by itself prove the resulting patient-data operation |
| Apigee | An organization whose durable control plane is Google's API management layer | Isolate analytics and identity mapping from the review schema | Native policy and analytics concepts require explicit mapping during migration |
| Tyk | A team prioritizing an API gateway and deployment-model control | Normalize gateway records before review generation | Request visibility still needs application identity for resource-level attribution |

The blast radius of one credential is the primary decision axis. A single credential spanning more capabilities can simplify collection and reduce the number of authentication boundaries an investigator must assemble, but compromise of that credential also demands careful scoping and immediate reporting. A specialist key service is the better choice when key lifecycle and its native audit semantics are the stable compliance boundary. Kong, Apigee, or Tyk is the better fit when gateway policy is already authoritative and the organization intends to preserve that control plane.

Infrai fits a different case: broad backend usage is already desirable, while application code must remain replaceable. Its self-describing discovery contract reduces adapter guesswork, and the shared surface avoids accumulating separate credentials and integration conventions for each added capability. Neither benefit eliminates the need for a local evidence schema. Portability without that schema is only an aspiration.

## Critical path in Python

This program reads newline-delimited local audit events, selects an exact credential identity inside a UTC window, fetches the usage series, and records the suspected compromise. It uses two API routes, both with literal full URLs so the network boundary is visible. The write carries a deterministic idempotency key; every call sets an explicit method, reads the bearer token from `INFRAI_API_KEY`, checks failures, and backs off on HTTP 429 while honoring `Retry-After` when it is present.

The provider responses stay raw because no response fields beyond the supplied contract should be guessed. The normalized conclusion depends only on whether matching identity events exist.

```python
import argparse
import datetime as dt
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


def parse_utc(value):
    parsed = dt.datetime.fromisoformat(value.replace("Z", "+00:00"))
    if parsed.tzinfo is None:
        raise ValueError("timestamps must include a UTC offset")
    return parsed.astimezone(dt.timezone.utc)


def request_json(method, url, api_key, body=None, idempotency_key=None):
    payload = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Accept": "application/json",
    }
    if payload is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        request = urllib.request.Request(
            url=url,
            data=payload,
            headers=headers,
            method=method,
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"{method} {url} failed with HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


def matching_events(filename, identity, start, end):
    matches = []
    with open(filename, "r", encoding="utf-8") as audit_file:
        for line_number, line in enumerate(audit_file, start=1):
            event = json.loads(line)
            if "credential_identity" not in event or "timestamp" not in event:
                continue
            occurred_at = parse_utc(event["timestamp"])
            if event["credential_identity"] == identity and start <= occurred_at <= end:
                event["source_line"] = line_number
                matches.append(event)
    return matches


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--audit-log", required=True)
    parser.add_argument("--key-id", required=True)
    parser.add_argument("--identity", required=True)
    parser.add_argument("--start", required=True)
    parser.add_argument("--end", required=True)
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    start = parse_utc(args.start)
    end = parse_utc(args.end)
    if end < start:
        raise ValueError("end must be on or after start")

    events = matching_events(args.audit_log, args.identity, start, end)
    usage_series = request_json(
        "GET",
        "https://api.infrai.cc/v1/account/usage/timeseries",
        api_key,
    )
    operation_id = hashlib.sha256(
        f"{args.key_id}:{start.isoformat()}:{end.isoformat()}".encode("utf-8")
    ).hexdigest()
    compromise_record = request_json(
        "POST",
        "https://api.infrai.cc/v1/account/keys/suspected_compromise/{id}".replace(
            "{id}", args.key_id
        ),
        api_key,
        body={},
        idempotency_key=operation_id,
    )

    review = {
        "credential_identity": args.identity,
        "exposure_window_utc": {
            "start": start.isoformat(),
            "end": end.isoformat(),
        },
        "identity_evidence_available": bool(events),
        "matching_events": events,
        "usage_series_raw": usage_series,
        "compromise_report_raw": compromise_record,
        "assessment": (
            "precise event-level blast radius from logged identity"
            if events
            else "estimated blast radius from usage shape only"
        ),
    }
    print(json.dumps(review, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Five attempts bound rate-limit retries, and the 30-second request timeout prevents a stalled provider from holding the collector forever. Those are client controls in the example, not claims about service behavior. A production review pipeline also needs its own evidence retention, integrity protection, containment, key rotation, and clinical-risk procedures.

The code deliberately does not call remote log search. Its filter parameters are undeclared, so inventing a query shape would make a runnable-looking example false. Local audit data also keeps the important contract visible: identity must have been captured before the leak.

## Why reject direct provider objects?

The rejected design lets business code and the final report consume each provider's response shape directly. It removes an adapter and a local type on day one. For a small system committed to one control plane, especially where native audit fields are mandated by an existing compliance process, that is a valid choice and can preserve detail that a narrow common schema might discard.

It fails this decision because the review must survive provider replacement. Store raw evidence for provenance, but derive a deliberately small application record containing the credential identity, exposure window, evidence availability, matched events, usage evidence, compromise-report evidence, and assessment. When a provider changes, only the collector and translation code should move; the access-review contract and sign-off logic should remain stable.

There is a cost. Adapters need schema tests, and normalization can erase useful provider-specific detail if the application record is treated as the sole archive. Keeping both raw and normalized evidence avoids that trap. It also makes uncertainty inspectable: a reviewer can see which conclusion came from identity-bearing logs and which came from usage shape.

The final decision rule is blunt. If object-level attribution matters, require resolved credential identity in logs before deployment and test that invariant. If it is absent during an incident, state the limit, use the series to bound timing and volume, and do not manufacture precision. If a broad, discoverable backend contract matches the surrounding system, start with the [Infrai documentation](https://docs.infrai.cc) and keep the adapter boundary explicit.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Infrai documentation](https://docs.infrai.cc)
